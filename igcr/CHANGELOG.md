# Changelog — qTap IGcr

All notable changes to this project are documented here.
Format: `Add`, `Fix`, `Update`, `Remove`, `Refactor`, `Security`.

---

## [2.59.0] — 2026-05-04
### Add
- **Comment sync from Instagram API**: New `POST /kdc/v1/instagram/comment/sync` endpoint and `KDC_QTAP_IGCR_Comments::sync_from_api()` fetch top-level comments and replies for a post, useful when webhook events were missed.
- **Comment hide/unhide on Instagram**: New `POST /kdc/v1/instagram/comment/hide` endpoint toggles `?hide=true|false` on the Graph API and tracks state via `_igcr_hidden` comment meta. Hide/Unhide buttons appear on each comment for admins; hidden comments dim and strike through.
- **DM `human_agent` messaging tag**: `KDC_QTAP_IGCR_API_Messaging::set_human_agent()` adds `messaging_type=MESSAGE_TAG` + `tag=human_agent`, extending the manual-reply window from 24 hours to 7 days. Wired into the inbox reply path.
- **Eye / eye-off icons** added to the SVG sprite for hide/unhide UI.
- **Theme-first link colors**: low-specificity `:where()` rules in `public-base.css` so theme styles win over the accent fallback.

### Update
- **Single post page redesign**: Cleaner Instagram-style header (avatar + username + IG glyph link, no Follow/dotted menu); caption shown without avatar for less clutter; action bar reorganized with inline like/comment counts; Sync Comments admin button added; Comment form gains avatar and the Post button hides until input has content; admin toolbar simplified to draft-publish only.
- **Single comments view**: Added Sync Comments button, View link to the original IG comment, Hide/Unhide on others' comments, "Own" indicator on the account's own comments, account-avatar on own replies; removed the comment heart and inline author name for an Instagram-faithful look.
- **Templates page**: Removed "Pull from Instagram" button (sync now handled per-account flow).

### Fix
- **Orphaned comment parents**: When a parent comment arrives via webhook *after* its reply, both the upsert path and the thread builder now back-fill `comment_parent` so threads render correctly without a manual re-sync.
- **Automation modal false dirty state**: Save button now clears any pending debounce timers (top-level + chain panels) so localStorage drafts can't be recreated after Save, eliminating spurious "unsaved changes" warnings on the next modal open.

## [2.58.0] — 2026-03-14
### Add
- **Message Templates admin page**: New submenu page under qTap IGcr for managing message templates (CRUD, sync to Instagram, pull from Instagram).
- **Template library JS**: Card-based grid UI with category filter tabs, inline editing, type-specific config fields (generic, button, quick reply, one-time notification).
- **Template selector in flows**: Registered `template` reply type in automation extension registry with category-grouped dropdown and inline preview.
- **Templates section in automation partial**: Shows template count with "Manage Templates" link to admin page.

### Fix
- **DM thread sorting**: Inbound webhook messages now touch `post_modified` so the conversation moves to the top of the thread list (matching outbound message behavior).

## [2.57.0] — 2026-05-04
### Add
- **Meta OAuth Connector Proxy** for third-party WordPress sites running the `kdc-igcr-wc` plugin. Centralizes Instagram Business Login on the qTap primary site so external sites don't each need their own Meta App.
  - REST endpoints under `/wp-json/kdc/v1/connector/oauth/*`: `start`, `callback`, `bootstrap`, `exchange`, `disconnect`.
  - TOFU site self-registration on first connect — proxy issues a per-site `site_key` + HMAC `site_secret`. Subsequent flows are HMAC-signed.
  - `igcr_connector_sites` and `igcr_connector_accounts` network tables (DB version → 2.2.0).
  - Daily cron `igcr_connector_refresh` — refreshes 60-day tokens within 10 days of expiry and pushes the new token to each external site, signed. 5-strike auto-deactivation.
  - Single static redirect URI to register with Meta: `/wp-json/kdc/v1/connector/oauth/callback`. Coexists with the existing in-house `/instagram/oauth/callback`.
- **Connector webhook relay**: Meta webhook entries for connector-owned IG accounts are forwarded to the originating external site, signed with the per-site secret (HMAC includes a sha256 of the entry payload). Local dispatch is skipped for relayed entries.
- **Connector data-deletion relay**: Meta data-deletion requests for connector accounts are mirrored to the originating external site over a signed POST. Per-site results are recorded on the deletion transient and surfaced in admin notices and email.
- **Token-at-transport encryption**: tokens leave the proxy AES-256-CBC encrypted with a key derived from the per-site secret, so the proxy's `AUTH_KEY` is never shared and intercepted tokens are unusable without the receiving site's secret.
- `docs/kdc-igcr-wc-connector-prompt.md` — self-contained implementation prompt for building the matching client side in the `kdc-igcr-wc` plugin (settings, encryption helpers, four REST receive routes, connect/disconnect UI).

## [2.56.5] — 2026-03-14
### Update
- **Plan Usage widget**: Redesigned to WooCommerce Status style — 2-column grid layout with colored status icons, progress bars, and plan badge header.
- **Post counting**: Only locally-created posts published to IG count against the plan limit; synced-from-IG posts shown as informational count ("X synced from IG").
- **Admin limit notices**: Styled with colored pill badges per resource, dashicons, and tinted backgrounds instead of plain text.

## [2.56.0] — 2026-03-14
### Update
- **Onboarding step 1**: Email collected before "Continue with Instagram" — stored via OAuth transient, used as WP user email and subsite admin email.
- **Username from IG**: After OAuth completes, WP `user_login` is updated to match the first connected Instagram username.
- **Onboarding step 2**: Email field removed (already collected in step 1). Account selector cards capped at `max-width: 480px`.
- **OAuth guest flow**: `create_or_find_wp_user()` uses onboarding email for real user email instead of `@igcr.local` placeholder.

## [2.55.5] — 2026-03-14
### Update
- **Onboarding**: "Connect Instagram" button now matches login button design — white background, accent color border, Instagram gradient glyph icon, "Continue with Instagram" label.

## [2.55.4] — 2026-03-14
### Fix
- **Instagram login button**: Use accent color (`--igcr-accent`) as 2px border on all login button variants (wp-login.php, WooCommerce, shortcode). Increased bottom margin on wp-login.php for better spacing.

## [2.55.3] — 2026-03-14
### Fix
- **Instagram login button**: Switched from Instagram gradient background to brand-guideline-safe white button with gradient glyph icon. Improved spacing between button and "or" divider on both wp-login.php and WooCommerce My Account.

## [2.55.2] — 2026-03-14
### Update
- **Instagram login priority**: "Continue with Instagram" button now renders above the default login form on both wp-login.php and WooCommerce My Account, with Instagram gradient styling and icon.
- **Hook changes**: wp-login uses `login_message` filter (renders above form) instead of `login_form` action (inside form); WooCommerce uses `woocommerce_login_form_start` instead of `woocommerce_login_form`.
- **Shortcode**: `[igcr_login_button]` now includes the Instagram icon.

## [2.55.1] — 2026-03-14
### Fix
- **Accent color picker**: Enqueue `wp-color-picker` script and style on Network Settings → Appearance tab so the WordPress core color picker renders correctly instead of a plain text input.

## [2.55.0] — 2026-03-14
### Refactor
- **Automation flows** extracted into self-contained partials (matching Create Post modal pattern):
  - `automation-settings-partial.php` — flows page (`/flows/`), replaces `flows-block.php`
  - `post-automation-modal.php` — single post automation modal, extracted from `single-igcr-post.php`
- Each partial guards its own prerequisites, enqueues its own CSS/JS, and is includable from any template
- Removed ~80 lines of conditional enqueuing from `public.php` — assets now load only when the partial renders
- Deleted `flows-block.php` (superseded)

---

## [2.54.0] — 2026-03-14
### Add
- **Plan Usage dashboard widget** — progress bars for accounts, posts, products, flows with color coding (green/amber/red)
- **Admin notices** on qTap IGcr pages when resources are at ≥80% (warning) or 100% (error) of plan limit
- **WooCommerce product edit** inline notice when product sync limit is reached — prevents enabling sync
- **Post editor notice** showing current post count vs plan limit on igcr_media post type
- **Create Post modal** counter badge in header showing `current / limit` (uses ∞ for unlimited plans)
- Create Post modal limit-reached banner when post publishing is blocked

---

## [2.53.0] — 2026-03-14
### Add
- **Plan limit enforcement** for posts, products, and flows — previously only accounts were enforced
  - Media sync stops creating new posts when `max_posts` reached
  - Publishing to Instagram blocked when post limit reached (`_igcr_status = 'limit_reached'`)
  - WooCommerce product sync silently refuses when `max_products` reached
  - Keyword rule, chain, and post template creation returns 403 when `max_flows` reached
- `Site_Plan::can_create()` + `count_resource()` convenience helpers for limit checks
- **Email collection** in onboarding flow — required field before subsite creation
  - Email stored as `user_email` on the WP user and `_igcr_onboard_email` in user meta
  - New subsite `admin_email` option explicitly set to the provided address
  - Client-side and server-side validation

---

## [2.52.0] — 2026-03-14
### Add
- **Permission Monitor** — proactive daily check of `GET /me/permissions` for all connected accounts; detects revoked scopes and logs changes
- **Reactive permission detection** — API Client detects Instagram permission errors (OAuthException codes 10, 190, 200) in real time and triggers an immediate scope refresh
- `granted_scopes` column on `igcr_accounts` table — stores the current set of granted Instagram permissions per account
- OAuth callback now fetches and stores `GET /me/permissions` at connect time as the baseline
- Non-dismissible admin notices for permission revocations — persists until admin acknowledges
- Activity log entries for all permission events (`event_type = 'permission'`) with full scope diff details

---

## [2.51.0] — 2026-03-14
### Security
- Removed hard-coded encryption key fallback in `KDC_QTAP_IGCR_Crypto` — now derives a site-unique key if `AUTH_KEY` is missing
- Redacted sensitive API response data from `error_log` calls in OAuth token exchange
- Added strict `validate_callback` (numeric string, max 20 chars) to all `account_id` REST params in DM endpoints

### Refactor
- Context-aware lazy loading: `load_dependencies()` now groups class files by request context (network admin, site admin, frontend, REST) — skips ~25 files on frontend requests
- REST endpoint classes deferred to `rest_api_init` callback instead of loading on every request

### Remove
- Deleted orphaned `KDC_QTAP_IGCR_Flow_Condition` class (never loaded, superseded by slot-based automation)
- Deleted unused legacy meta box views (`reel-meta-box.php`, `story-meta-box.php`, `highlight-meta-box.php`)
- Added `@deprecated` annotation to legacy CPT classes (Reel, Story, Highlight)

### Add
- `readme.txt` for WordPress Plugin Review compliance
- `LICENSE` file (GPL-2.0-or-later)
- `docs/features.md` — comprehensive feature summary (18 features)
- `docs/marketing-prompts.md` — AI prompt templates for marketing content
- `docs/support-rag.md` — support knowledge base by persona (Super Admin, Admin, Editor, Shop Manager)
- `docs/api-docs.md` — complete REST API reference for APIdog/GitBook

---

## [2.50.0] — 2026-03-14
### Refactor
- Network Settings page split into 4 tabs using WordPress native `nav-tab-wrapper`: **App**, **Accounts**, **Appearance**, **Advanced**
- Each tab has its own `<form>` and saves independently — only that tab's fields are written, preserving all other settings
- Tab state preserved in URL (`?tab=xxx`) and on save redirect
- `KDC_QTAP_IGCR_Network_Settings::save()` now accepts optional `$tab` parameter with per-tab field mapping

---

## [2.49.0] — 2026-03-14
### Add
- Network setting: **Deletion / Deauth Behaviour** with 3 modes:
  - **Delete data** — permanently remove account, tokens, user meta; send email + admin notice
  - **Deactivate only** — soft-delete (mark inactive) but keep data; send email + admin notice
  - **Do nothing** — log the request only; no email, no notice, no data changes
- Setting displayed on Network Admin → App Settings, saved in `igcr_settings['deletion_mode']`

---

## [2.48.1] — 2026-03-14
### Add
- Non-dismissible admin notice (site + network admin) on data deletion events — persists until admin clicks "Acknowledge & dismiss" (nonce-protected)
- Email notification sent to site admin + network admin with full deletion details: account, status, confirmation code, and explanation of what was/wasn't deleted

---

## [2.48.0] — 2026-03-14
### Add
- Instagram Data Deletion callback endpoint (`POST /kdc/v1/instagram/data-deletion`) — required by Meta for app review
  - Verifies Meta's `signed_request` via HMAC-SHA256 using the app secret
  - Deletes the Instagram account row and cleans up WordPress user meta (`_igcr_ig_user_id`, `_igcr_ig_username`, `_igcr_ig_name`)
  - Returns `confirmation_code` + status URL as Meta requires
  - Status check endpoint (`GET /kdc/v1/instagram/data-deletion/status?code=...`) for Meta to verify completion
  - Logs deletion events to Activity Log
- Data Deletion Request URL displayed on Network Admin → App Settings page with copy button

---

## [2.47.0] — 2026-03-14
### Add
- "Continue with Instagram" button on `wp-login.php` login form (hooked to `login_form`)
- "Continue with Instagram" button on WooCommerce My Account login form (hooked to `woocommerce_login_form`, only when WooCommerce active)
- Both buttons respect `redirect_to` / My Account return URL after successful login

---

## [2.46.6] — 2026-03-14
### Fix
- Instagram Login redirect_uri now routes through primary site (`ig.cr/?igcr_login=1`) instead of subsite URL — fixes "Invalid redirect_uri" on subsites like `kdc.ig.cr`
- Switched login transients from `set_transient`/`get_transient` to `set_site_transient`/`get_site_transient` (network-wide) so callback on primary site can read state set by subsite

---

## [2.46.5] — 2026-03-14
### Remove
- Removed "Instagram" column from WooCommerce products list table — not useful without catalog sync

---

## [2.46.4] — 2026-03-14
### Fix
- FAB panel now uses absolute positioning relative to trigger button instead of flexbox — prevents overflow off-screen
- Panel opens above trigger (bottom-right/bottom-left) or below trigger (top-right/top-left) with correct alignment

---

## [2.46.3] — 2026-03-14
### Fix
- FAB panel is now position-aware: opens above/below and left/right aligned based on where the button is dragged (uses `data-fab-pos` attribute with `column-reverse` for top placement)
- Fixed dark/light mode icon colors — removed hardcoded expanded-state styles that were always dark, now adapts via `prefers-color-scheme` and `data-igcr-theme`
- Panel and menu items use explicit color values instead of CSS vars (FAB renders outside igcr block wrappers)

---

## [2.46.2] — 2026-03-14
### Fix
- FAB responds to browser `prefers-color-scheme` — light mode: dark bg (`#0D0F14`), light icon (`#f5f5f5`), dark shadow; dark mode: light bg (`#f5f5f5`), dark icon (`#0D0F14`), white shadow
- Orange accent constant at `#f37521`
- Panel and menu items also adapt to dark mode

---

## [2.46.1] — 2026-03-14
### Fix
- Profile tab active indicator: thicker `2px` top border, added `!important` override to prevent theme bottom-border bleed

---

## [2.46.0] — 2026-03-14
### Add
- Floating Action Button (FAB) menu — IGcr brand icon appears on non-plugin pages for quick access to Account, Automation, Create Post, Profile, and Orders (WooCommerce)
- FAB is draggable with position persistence via localStorage, adapts to dark mode, and hides on pages with existing plugin navigation
- Self-contained partial (`fab-menu.php`) enqueues own CSS + JS
- Added `plus-square`, `user`, and `shopping-cart` icons to SVG sprite

---

## [2.45.0] — 2026-03-14
### Update
- Profile layout closer to Instagram.com: tab icons 12px → 24px, active indicator on top border, full-width action buttons, 4px grid gap
- Action button icons 14px → 16px for better proportions
- Tab nav border moved from bottom to top (matching Instagram.com)
- Mobile: tab icons scale to 20px with labels visible

---

## [2.44.3] — 2026-03-14
### Refactor
- Extract Create Post modal into self-contained partial (`create-post-modal.php`) that enqueues its own CSS + JS — eliminates HTML duplication across profile block and single post templates
- Both `account-profile-block.php` and `single-igcr-post.php` now use a single `include` line
- Remove scattered asset enqueues from profile block render and public class (the partial handles it)

---

## [2.44.2] — 2026-03-14
### Fix
- Replace old Create Post modal in profile block (`account-profile-block.php`) with new multi-step modal (Select → Compose → Automate) — was still using legacy tabs/form layout

---

## [2.44.1] — 2026-03-14
### Fix
- Load `public-profile.css` on single IG post pages so the Create Post modal is properly styled (dropzone, two-panel compose, automation step)

---

## [2.44.0] — 2026-03-14
### Refactor
- **JS code splitting**: Split monolithic `public.js` (1856 lines) into 7 task-specific fragments — `public-utils.js`, `public-single-post.js`, `public-profile.js`, `public-create-post.js`, `public-dm-inbox.js`, `public-sse.js`, `public-post-grid.js`
- Each fragment is enqueued only where needed (block render callbacks or conditional page checks), reducing JS payload on most pages by ~95%
- Shared utilities exposed via `window.igcr` namespace; no build step required

---

## [2.43.0] — 2026-03-14
### Update
- **DM inbox redesign**: Instagram.com-inspired messaging UI with full design token migration (light/dark mode), accent-colored outbound bubbles, active thread indicator
- **Automation message badges**: Messages sent via automation flows now display a "Sent via [Flow Name]" badge with zap icon, making automation a visible hero feature
- **Token consistency**: Replaced all hardcoded pink gradients and colors in messages CSS with `--igcr-accent` and semantic tokens

---

## [2.42.0] — 2026-03-14
### Add
- **Multi-step Create Post modal**: Instagram-style 3-step flow — Select media → Compose (two-panel preview + fields) → Automate
- **Drag-and-drop media upload**: Drop photos/videos onto the upload zone with smart type inference (1 video = reel, 1 image = post, 2+ images = carousel)
- **Instagram API extras**: Location, collaborators (up to 3), alt text, and share-to-feed toggle passed through to Instagram Graph API
- **Post-level automation at creation**: Set up comment auto-replies and keyword rules before publishing, with template shortcuts (More Info, Link in DM, Product Link, Payment Link)
- **Carousel management**: Thumbnail strip with add/remove, prev/next navigation, dot indicators in the compose preview
- **Discard protection**: Confirm dialog on close/escape when media or caption is entered; overlay click disabled for the create post modal

---

## [2.41.0] — 2026-03-14
### Add
- **Draft posts**: Save posts as WordPress drafts before publishing to Instagram — set up automation flows on unpublished content
- **Save as Draft button**: Create post form now has a "Save as Draft" option alongside "Publish to Instagram"
- **Publish Draft button**: Single post view shows "Publish to Instagram" for draft posts, publishing and updating the WP post in one step
- **Draft visibility**: Admins see draft posts in the profile grid with a "Draft" badge and dashed border; public visitors only see published posts
- **REST endpoint**: `POST /kdc/v1/instagram/publish-draft` publishes an existing WP draft to Instagram

---

## [2.40.0] — 2026-03-14
### Add
- **Automation modal protection**: Modal only closes via X button; overlay clicks and Escape key show unsaved-changes confirmation when cards are dirty
- **Version history**: Every automation save creates a revision (max 20); "History" button on each card lets you browse and restore previous configs
- **Offline resilience**: Saves queue in localStorage when offline, auto-sync to server on reconnect with status notification
- **Page unload warning**: Browser warns before navigating away with unsaved automation changes
### Fix
- All Automate trigger buttons now properly open the modal (previously only the first one worked)
- Modal overlay cursor changed to default (was misleadingly pointer)

---

## [2.39.0] — 2026-03-13
### Add
- **Site Cloner**: Extracted template cloning into dedicated `KDC_QTAP_IGCR_Site_Cloner` class — serialized-data-safe URL replacement (`search_replace_deep`), media file copying, nav menu/widget/FSE block cloning, legal page date cleanup, post-clone verification audit
- **Network Admin clone**: `assign_to_user()` now clones template site content (previously only self-serve onboarding did)
- **Onboarding progress stepper**: Visual step-by-step progress UI during site creation (Creating → Store → Instagram → Ready)
- **Wizard suppression**: WooCommerce, The Events Calendar, and qTap onboarding wizards auto-suppressed on cloned sites
### Fix
- Cloning no longer copies revisions, edit locks, or transient data from the template site
- Bare domain and JSON-escaped URLs now replaced during clone (prevents stale template references)

---

## [2.38.8] — 2026-03-13
### Fix
- Onboarding clone now remaps page IDs inside `wp_navigation` blocks — previously only domain was replaced, leaving stale source-site IDs that resolved to wrong URLs
- Instagram social login assigns `customer` role (instead of `subscriber`) when WooCommerce is active

---

## [2.38.7] — 2026-03-13
### Fix
- Block theme template override now works (Twenty Twenty-Five etc.) — directly overrides `$_wp_current_template_content` global instead of late `get_block_templates` filter that fired after resolution

---

## [2.38.6] — 2026-03-13
### Fix
- Never crop media on single igcr_media view — changed `object-fit` from `cover` to `contain` for images, videos, and carousel slides so media always displays at full natural size
- Removed forced 1:1 aspect ratio on mobile media column

---

## [2.38.5] — 2026-03-13
### Add
- Minimal single template for `igcr_media` on non-IGcr themes — renders only header + content + footer, bypassing theme's single-post layout (sidebars, author boxes, post meta)
- `.igcr-single-main` CSS reset to prevent theme layout bleed

---

## [2.38.4] — 2026-03-13
### Add
- Redirect `/p/{slug}` 404s to profile page with "Media not found" warning notice instead of showing a blank 404
- Warning notice style (`.igcr-notice--warning`) in profile CSS

---

## [2.38.3] — 2026-03-13
### Fix
- Single post media black bars — replace fixed `height` with `min-height: 480px` / `max-height` so card adapts to image aspect ratio; switch `object-fit` from `contain` to `cover`

---

## [2.38.2] — 2026-03-13
### Fix
- Single igcr_media post CSS and automation JS not loading — `is_singular('post')` in public asset enqueue updated to `is_singular( KDC_QTAP_IGCR_CPT_Post::POST_TYPE )`

---

## [2.38.1] — 2026-03-13
### Remove
- Posts, DMs, and Flows submenu items from wp-admin sidebar — all editing is frontend-only
- `show_ui` on `igcr_media`, `igcr_dm`, `igcr_flow` CPTs — backend edit screens no longer accessible
- Admin column hooks, editor sidebar, publish gate, and account filter (dead code with `show_ui=false`)
- `set_parent_file` / `set_submenu_file` filters from site admin (no longer needed)

---

## [2.38.0] — 2026-03-13
### Refactor
- Migrate Instagram media from native `post` type to dedicated `igcr_media` CPT across all 19 touchpoints (media sync, flow triggers, blocks, publish endpoint, network admin, uninstall)
- Register `igcr_media` CPT with full labels, REST API support, `p/{shortcode}` permalinks, and `igcr_content_type` taxonomy
- Add one-time migration tool in Network Admin → Tools to convert existing synced posts to `igcr_media`

---

## [2.37.0] — 2026-03-13
### Add
- Modular automation JS architecture: extension registry pattern (`window.igcrAutoExt`) with `types`, `renderers`, `binders`, `collectors`, `previewers` hooks
- Product Carousel reply type extracted to `automation-products.js`, conditionally loaded only when WooCommerce is active
- Events reply type (`event_carousel`) in `automation-events.js`, conditionally loaded when kdc-qtap-events or Events Ticket Plus is active — supports Card (1 event) or Carousel (2-10 events) layout with event picker, Buy/Info buttons, and dynamic variables
- Events REST endpoint (`GET /kdc/v1/instagram/events`) serving event data from kdc-qtap-events or Tribe Events
- `send_event_carousel` flow action: builds Instagram Generic Template elements from event data with image, title, date/venue subtitle, and Buy Ticket / Event Details buttons

### Refactor
- `automation-settings.js` and `post-automation.js` now use `getReplyTypes()` (base + extensions) instead of hardcoded `REPLY_TYPES` array
- Extension scripts share utilities via `window.igcrAutoUtils` (`esc`, `api`, `apiBase`, `charCounterHTML`, `bindCharCounter`)

## [2.36.1] — 2026-03-13
### Update
- Accent color picker in Network Settings now uses the WordPress core `wpColorPicker` widget instead of a custom `<input type="color">` + hex text field

## [2.36.0] — 2026-03-13
### Update
- Accent color CSS tokens now derive a full hierarchy from the single Network setting: `--igcr-accent`, `--igcr-accent-hover`, `--igcr-accent-active`, `--igcr-accent-soft`, `--igcr-accent-light`, `--igcr-accent-border`, `--igcr-accent-rgb`
- Default accent falls back to Instagram's primary blue (`#0095F6`) when no color is configured
- Fixed stale `--igcr-primary` references in automation CSS → unified to `--igcr-accent`

## [2.35.0] — 2026-03-13
### Add
- Instagram post block: grid column options (3 or 6, default 6)
- Instagram post block: loading style setting — Autoscroll (infinite), Load More button, or Pagination (default: Autoscroll)
- Instagram post block: total posts limit (All / 12 / 24 / 48 / 96 / 120) with per-page control (6 / 12 / 24 / 48, default 12)
- REST endpoint `GET /kdc/v1/instagram/posts` for AJAX-powered pagination

---

## [2.34.1] — 2026-03-13
### Fix
- Chain responses now always saved on Save click — removed draft-gate that skipped chains due to debounce race
- Fix button row scoping: main/DM panel config no longer collects buttons from nested chain panels

---

## [2.34.0] — 2026-03-13
### Add
- `qtap/igcr-menu` block — injects account-aware nav items (Accounts, Messages, Flows, Create) into the theme sidebar and mobile nav via `igcr_theme_nav_items` and `igcr_theme_mobile_items` filters
- Menu items only appear when the subsite has a connected Instagram account

### Refactor
- Move plugin-specific navigation responsibility from theme to plugin — theme now provides filter hooks, plugin decides what to show

## [2.33.2] — 2026-03-13
### Fix
- Force WP Media Library modal to light mode — theme dark mode no longer breaks header/label colors

---

## [2.33.1] — 2026-03-13
### Fix
- DM preview: media types (image/video/audio/file) no longer double-wrapped in bubble+attachment — image previews render correctly
- Replace hardcoded SVG stroke colors (#999, #bbb) with `currentColor` for dark mode compatibility in card/product previews
- Replace hardcoded CSS colors (#c0392b, #ccc, #d32f2f, #f0f0f0, #f0f2f5) with design system tokens across automation settings

---

## [2.33.0] — 2026-03-13
### Add
- Prominent "Automate" CTA button in single post top bar — accent-colored, positioned opposite "Back to Profile"
- Previous/next post navigation arrows flanking the single post card, with keyboard left/right arrow support
- Videos autoplay muted by default; user unmute preference persisted via session cookie

---

## [2.32.7] — 2026-03-13
### Fix
- Accounts block buttons use plugin `.igcr-action-btn` styling instead of WP `wp-element-button` class; move shared button styles to `public-base.css`; replace hardcoded `#f0f0f0`/`#ccc` with design tokens for dark mode compatibility

---

## [2.32.6] — 2026-03-13
### Fix
- Disconnect modal buttons use plugin `.igcr-modal-btn` styling instead of WP `.button` class for dark/light mode consistency

---

## [2.32.5] — 2026-03-13
### Update
- Redesign admin settings panel: subtle bordered card with `--igcr-bg-subtle` background, all hardcoded colors replaced with design tokens for dark/light mode, reconnect and disconnect buttons use plugin styling instead of WP `.button` class

---

## [2.32.4] — 2026-03-13
### Update
- Default post grid to 6 columns, widen profile wrap to 1200px max-width, 3-col fallback on mobile

---

## [2.32.3] — 2026-03-13
### Fix
- Move admin settings panel to open directly below the action buttons instead of after the post grid

---

## [2.32.2] — 2026-03-13
### Fix
- Sync notice uses theme-aware CSS tokens for dark/light mode compatibility

---

## [2.32.1] — 2026-03-13
### Update
- Profile block: swap display name above stats row (username → name → stats), support line breaks in bio via `nl2br`, restore icons on action buttons, move tab underline to bottom

---

## [2.32.0] — 2026-03-13
### Update
- **Instagram.com-style profile page**: reworked `/profile/{username}/` to match Instagram.com's flat, minimal design — CSS grid header with 150px avatar, 20px username (no @ prefix), prominent stats row (posts → followers → following), display name + bio below stats inside info column, text-only rounded-rectangle action buttons, icon-only tabs with top-border active indicator, sharp-cornered post grid, dark-mode-safe CSS tokens throughout
- Responsive mobile layout: 77px avatar, centered stacked layout, tab labels visible, 3-col grid preserved

---

## [2.31.1] — 2026-03-13
### Add
- **`#create` hash anchor**: navigating to any page with the Create Post modal and `#create` in the URL auto-opens the modal — enables nav menu "Create" link (`/profile/#create`)

---

## [2.31.0] — 2026-03-13
### Add
- **Active account session**: cookie-based session (`igcr_active_account_{blog_id}`) remembers the last-viewed Instagram account across profile, messages, flows, and accounts pages
- `get_active()` / `set_active()` helpers on `KDC_QTAP_IGCR_Instagram_Account`
- DM and flows account pickers persist selection to cookie via JS
- `/profile/` redirect for multi-account sites now goes to the active account instead of `/accounts/`

---

## [2.30.5] — 2026-03-13
### Update
- **Single post layout — Instagram-style height-anchored card**: card fills ~90% viewport height (`min(calc(100vh - 96px), 898px)`), media fills left column with `object-fit: contain`, content panel width scales via `clamp(340px, 33vw, 500px)` for comfortable reading
- Added tablet (736–1024px) and ultra-wide (≥1600px) responsive breakpoints
- Mobile: stacked layout with square-cropped media (`object-fit: cover`)

---

## [2.30.4] — 2026-03-13
### Fix
- **Single post layout**: removed max-width constraint and side padding from `.igcr-single-wrap` so the post card fills the available width without dead whitespace on desktop

---

## [2.30.3] — 2026-03-13
### Fix
- **Single post media sizing**: image now dictates card height naturally (removed fixed `min-height` and `object-fit: contain`), media fills full width and the card grows to match — no wasted black space

---

## [2.30.2] — 2026-03-13
### Update
- **Single post layout — Instagram-style max width**: wrapper expanded from `935px` to `1200px`, media column now takes all available space (`1fr`) with side panel capped at `405px`

---

## [2.30.1] — 2026-03-13
### Refactor
- **Standardized modal system**: consolidated all modal CSS (automation, create post, disconnect) into a single shared base in `public-base.css` — `.igcr-modal`, `.igcr-modal-overlay`, `.igcr-modal-box`, `.igcr-modal-header`, `.igcr-modal-close`, `.igcr-modal-content`
- Refactored automation modal HTML to use the shared `.igcr-modal` pattern (was custom `.igcr-auto-modal` with `:has()` trick)
- Unified JS modal handlers: single overlay click, close button, and Escape key handler covers all modals
- Removed ~80 lines of duplicate modal CSS from `public-profile.css` and `public-single-post.css`

---

## [2.30.0] — 2026-03-13
### Update
- **Consistent border-radius**: replaced all hardcoded `border-radius` values across public CSS with design tokens — `--igcr-radius-sm` (6px), `--igcr-radius` (8px), `--igcr-radius-lg` (12px), `--igcr-radius-full` (pill). Covers profile, messages, single post, onboarding, and automation styles
- **Dark mode hardening**: replaced remaining hardcoded `#fff`, `#ddd`, `#eee`, `#f0f0f0`, `#fafafa` colors in messages and onboarding CSS with CSS custom properties for full dark/light/system support

---

## [2.29.1] — 2026-03-13
### Update
- **Single post UI polish**: disabled heart and bookmark action icons (dulled, non-interactive), highlighted Automate button with accent color, added Create Post button and modal to single post page, admin toolbar now flex row layout

---

## [2.29.0] — 2026-03-12
### Update
- **Standardized layout widths**: all block container `max-width` aligned to `935px` — automation (was 720px), profile (was 960px), messages (was 900px/1000px), single post (already 935px)
- **Standardized responsive breakpoints**: mobile `735px` (was 560/600/640/680), desktop split `876px` (was 768), tablet grid `875px` (was 900)
- Fixed `.igcr-account-card` hardcoded `#fff` background for dark mode

---

## [2.28.1] — 2026-03-12
### Fix
- **Automation dark mode**: replaced all hardcoded `#fff`/`#ddd`/`#e5e7eb` colors in `public-automation.css` with CSS custom properties — account selector, card backgrounds, toggle sliders, form inputs, selects, textareas, DM preview phone frame, chat area, received bubbles, card previews, quick reply chips, carousel items, variant rows, button-row fields, DM panel
- **Save button alignment**: save button now spans full card width when card body is flex (desktop)
- **Automate modal dark mode**: all inputs, selects, and textareas inside the post-level automation modal now respect the subsite color scheme

---

## [2.28.0] — 2026-03-12
### Update
- **Single post UI polish (round 2)**: automation panel moved to fixed-position modal overlay with backdrop (CSS `:has()` driven), caption reformatted as comment-like row with avatar + timestamp, heart icon buttons added to all comments and replies, content column scroll fix (`flex: 1 1 0; min-height: 0`), card border-radius reduced to 4px

---

## [2.27.1] — 2026-03-12
### Fix
- **Dark mode color refinement**: replaced desaturated blues (#5a8bb8) with vibrant #0095F6, fixed muted text (#525252 → #737373), aligned border-light (#1a1a1a → #262626), updated plugin base dark tokens

---

## [2.27.0] — 2026-03-12
### Add
- **Single post UI redesign**: bordered card layout with media left / content right, header strip (avatar + username + Follow + menu), flat comment list, action bar (heart/comment/share/bookmark), like count, date, inline comment form
- "More posts from username" 3-col grid below the single post card
- Engagement data sync: `like_count` and `comments_count` from Instagram Graph API stored in post meta
- "View all N replies" collapsible toggle for comment reply threads
- **Dark mode / Light mode / System**: per-subsite `igcr_color_scheme` setting with CSS custom property overrides scoped to `[data-igcr-theme]` attribute
- Site Settings admin page under qTap IGcr menu for color scheme selection
- `bookmark` and `more-horizontal` icons added to SVG sprite

### Update
- Single post CSS fully rewritten for card layout with responsive mobile breakpoint
- Comment display changed from chat bubbles to flat-list style
- SSE real-time comment handler updated for new flat comment markup
- Comment form moved to inline footer position (textarea + "Post" button)

---

## [2.26.0] — 2026-03-12
### Add
- **IGcr theme v1.1.0**: Custom login page template (`page-login`) — two-column layout with admin-editable hero (image + copy) on left, login form on right
- Login form block (`igcr/login-form`): WP-native auth, "Connect with Instagram" button when plugin active, forgot password link, registration link
- `wp-login.php` redirects to custom login page when a published page at `/login/` exists; failed logins redirect back with error message
- Site footer block (`igcr/site-footer`) with WP-managed Footer menu + copyright — added to all templates
- WP-managed Header menu rendered in sidebar bottom and mobile header (Cart, Checkout, Login links)
- Dark mode support for login page and header/footer menu components

---

## [2.25.0] — 2026-03-12
### Add
- **IGcr theme** (v1.0.0): WordPress block theme at `wp-content/themes/igcr/` — fixed sidebar navigation (desktop), mobile header+footer chrome, dark mode via `prefers-color-scheme`, role-based nav items, search overlay
- Theme is fully independent of the IGcr plugin — works standalone as a generic blog theme; plugin features are additive (nav items, profile URLs, synced-post detection)
- Single post conflict resolution: `igcr-synced-post` body class hides theme title/image/comments when plugin's `the_content` filter renders the two-column IG media layout
- Theme ships its own SVG icon sprite and `igcr_theme_icon()` helper — no dependency on plugin's icon system

---

## [2.24.2] — 2026-03-12
### Fix
- **Post automation UI**: Force card body to stack vertically inside the narrow single-post column — preview phone was overlapping form fields when the side-by-side flex layout activated at viewport width > 768 px

---

## [2.24.1] — 2026-03-12
### Refactor
- **JS deduplication**: Unified `bindChainPanel` and `bindMenuChainPanel` into a single parameterized function; inlined `collectChainConfig` wrapper into direct `collectPanelConfig` calls (45 lines removed)

---

## [2.24.0] — 2026-03-12
### Refactor
- **CSS splitting**: Split monolithic `public.css` (4,361 lines) into 6 page-scoped partials (`public-base.css`, `public-profile.css`, `public-single-post.css`, `public-messages.css`, `public-onboarding.css`, `public-automation.css`) — each loaded conditionally via block render callbacks or page-type checks
- **PHP endpoint splitting**: Split `class-kdc-qtap-igcr-flow-endpoint.php` (1,291 lines) into 3 focused endpoint classes: slot/chain CRUD, post-level automation, and persistent menu/ice breaker routes
- **Dead code cleanup**: Removed `node_handles_flows()` method (hardcoded to `return false`) and all 6 dead guard blocks from flow triggers
### Security
- **WooCommerce nonce**: Added explicit nonce verification to `save_product_fields()` in the WooCommerce integration

---

## [2.23.0] — 2026-03-12
### Update
- **Manual Save for automations**: Automation card editing no longer autosaves on every keystroke — changes are cached in localStorage and only persisted when the user clicks the per-card Save button
- **localStorage draft persistence**: Unsaved automation edits survive page reloads and are restored automatically until explicitly saved or discarded
- **Chain panel drafts**: Chain response edits are also cached locally and saved together with the parent card's Save button

---

## [2.22.1] — 2026-03-12
### Fix
- **Product picker not working on post automation page**: Replaced placeholder with fully functional product picker grid (search, variation expand, selection) for the DM follow-up Product Carousel reply type. Also adds pay/info button label fields with variable pill insertion and character counters

---

## [2.22.0] — 2026-03-12
### Fix
- **Slot-based automations not running**: `node_handles_flows()` was blocking PHP from dispatching slot-based flows (comment auto-reply, keyword rules, DM follow-ups, etc.) because Node only supports graph-based flows. PHP now always handles slot-based automations — no meta key overlap with Node's graph-based system
- **DM follow-up using wrong recipient format**: Comment-triggered DM follow-ups now use the private reply API (`recipient.comment_id`) instead of `recipient.id`, enabling DMs to commenters who haven't previously messaged the page
- **All DM actions use comment context**: Refactored `make_messaging()` helper in Flow Action to auto-set comment context when `ig_comment_id` is in the trigger context

---

## [2.21.1] — 2026-03-12
### Fix
- **Drag reorder not working**: Drag initiation now restricted to grip handle only — interactive elements (inputs/selects) no longer hijack drag events
- **Chain response toggle not opening for Ice Breakers**: `bindMenuChainToggle` now finds both menu and ice breaker rows; `dirtyFn` correctly wired through `bindMenuChainPanel` so ice breaker chain edits trigger the right save callback

---

## [2.21.0] — 2026-03-12
### Add
- **Drag-to-reorder**: Persistent Menu and Ice Breaker rows are now drag-sortable via a grip handle
- **Ice Breaker chain responses**: Ice breaker items now support "Configure chain response" — same as Persistent Menu postback items

### Fix
- **Persistent Menu not updating on Instagram**: `set_persistent_menu` and `set_ice_breakers` now use JSON `Content-Type` via new `post_json()` client method — form-urlencoded was not correctly serializing nested `call_to_actions` arrays for the `messenger_profile` endpoint
- **URL field styling in Persistent Menu**: Added `input[type="url"]` to CSS selectors so URL fields match other input styles
- **URL input validation**: Button value fields now use `type="url"` for `web_url` buttons

---

## [2.20.1] — 2026-03-12
### Fix
- **Product picker search**: Cards now properly hide/show — CSS `display: flex` was overriding the `hidden` attribute; added explicit `[hidden]` rule
- **Variable product expand**: Same `hidden` override fix — variant cards now toggle visibility correctly
- **URL field styling in Persistent Menu**: Added `input[type="url"]` to menu row and button row CSS selectors so URL fields match other field styles

---

## [2.20.0] — 2026-03-12
### Add
- **Product picker search**: Instant search bar filters products by title
- **Variable product support**: Variable products show an expand chevron to reveal variation cards; selecting a variation highlights the parent with a dashed border
- **URL input validation**: Button value fields and menu URL inputs now use `type="url"` for web_url entries

### Update
- **Product cards**: Compact grid with title-only (truncated with ellipsis, full title on hover tooltip), removed price from cards

---

## [2.19.1] — 2026-03-12
### Remove
- **Payment Link reply type**: Removed `igpe_link` as a standalone reply type — payment links are now handled exclusively via the Payment Link button type within Buttons/Card reply types

---

## [2.19.0] — 2026-03-12
### Refactor
- **Unified reply panel code**: Consolidated 3 duplicate implementations (main/DM, chain, menu chain) into shared `bindPanelFields()`, `collectPanelConfig()`, and a unified `renderTypeFields()` — removed ~320 lines of duplicate code
- Deleted `renderChainTypeFields`, `chainButtonsListHTML`, `bindMenuChainPanelEvents`, `bindMenuChainPanelFields`, `bindMenuChainButtonRow`, `bindMenuChainAddBtn`, `bindMenuChainMediaPickers`
- `bindButtonRow` now accepts an `onDirty` callback for context-appropriate save behavior

## [2.18.0] — 2026-03-12
### Add
- **Payment Link button type**: Button type selector now has URL, Postback, and Payment Link options; Payment Link buttons accept an amount value and generate an ig.pe link at send time; preview shows `₹{amount}` next to the label
- **Product Carousel in chain panels**: Chain response panels (both automation cards and persistent menu) now fully support Product Carousel — product picker, pay/info button labels with dynamic variable pills and char counters
### Fix
- **Shared button formatter**: All button formatting (card, buttons, payment link) uses a shared `format_buttons()` / `format_button()` PHP method that handles URL, Postback, and Payment Link types consistently

## [2.17.0] — 2026-03-12
### Add
- **Payment Link buttons**: Payment Link reply type now supports up to 3 buttons (URL/Postback) with postback chaining; when buttons are configured, the payment link is sent as a button template instead of plain text; buttons shown in live preview below the payment block

## [2.16.1] — 2026-03-12
### Fix
- **Payment Link**: Removed message variants (not applicable); reordered fields to Message → Amount → Slug
- **Post automation**: Removed DM preview panel that was breaking the layout on post-level automation cards

## [2.16.0] — 2026-03-12
### Add
- **Dynamic product carousel labels**: Pay and Info button labels now support `{{product_title}}`, `{{product_amount}}`, and `{{product_category}}` placeholders — interpolated per-product at send time; clickable variable pills below each label field insert tags at cursor
- **Button label character counter**: All button label inputs (card, buttons, carousel, post-automation) now show a live `N/20` character counter; counter turns red when over limit
- **Truncation with ellipsis**: Button titles and carousel labels exceeding 20 characters are truncated with `…` suffix instead of a hard cut; product title/subtitle truncated at 80 chars with same suffix

## [2.15.3] — 2026-03-12
### Fix
- **Persistent menu chain panel**: Now uses the same dynamic `renderChainTypeFields()` as automation cards — changing reply type swaps in the correct fields (buttons list, media picker, card fields, etc.) instead of always showing a static message textarea

## [2.15.2] — 2026-03-12
### Update
- **Persistent menu rows**: Restructured to match button row pattern — type select (URL/Postback) first, then label, then value; postback items show "Configure chain response" toggle below the row with inline chain panel

## [2.15.1] — 2026-03-12
### Fix
- **Postback chain layout**: Chain config panel now renders as a new row below the button instead of inline beside it; chain preview renders within the same phone frame instead of creating a duplicate Instagram preview

## [2.15.0] — 2026-03-12
### Add
- **Postback chaining**: Configure multi-step conversational menus (menu → sub-menu → action) directly within automation cards; postback buttons on `card` and `buttons` reply types get a "Configure chain response" toggle that expands an inline config panel for the chained response; chains are stored as real keyword rules (matched by payload), cascade-deleted and cascade-toggled with the parent; live preview shows the full conversation tree with tap simulations; up to 3 levels deep

## [2.14.0] — 2026-03-12
### Add
- **Live DM preview**: Expanded automation cards on desktop now show a phone-frame DM preview on the right side, updating live as you type; supports all 10 reply types (text, image, video, audio, file, card, buttons, media share, payment link, product carousel) plus comment reply and DM follow-up previews; hidden on mobile

## [2.13.0] — 2026-03-12
### Add
- **Dynamic flows URL**: Flows page now supports `/flows/{ig-username}` for direct-link account selection; account picker updates URL via pushState
- **Post automations list**: Flows page now includes a paginated list of posts with post-level automation rules, linking directly to each post

## [2.12.1] — 2026-03-12
### Fix
- **Message variants visibility**: Variants field now only shown for reply types that use a message (text, buttons, quick reply, payment link); hidden for card, image, video, audio, file, media share, and product carousel

## [2.12.0] — 2026-03-12
### Add
- **Post-level automation**: Each synced Instagram post now has an inline "Automate" panel with per-post Comment Auto-Reply and Keyword Rules that take priority over global comment automation
- **Automation templates**: One-click presets (Intent of Purchase, Thank You Reply, DM Request, Link Request) that create keyword rules pre-filled with keywords and suggested messages
- **Post-level trigger cascade**: Post keyword rules → post comment reply → global comment_auto_reply; first match wins

### Fix
- **Card reply type**: Default Action URL field now renders full-width matching other fields
- **Button type placeholder**: Switching between URL/Postback now correctly updates the value placeholder

## [2.11.0] — 2026-03-12
### Add
- **Opt-out enforcement**: Users who send "stop"/"unsubscribe" are now blocked from all automated DMs and comment replies (except opt-in/opt-out responses); sending "start"/"subscribe" re-enables delivery
- **User resolution in comment/mention triggers**: Comment, live-comment, and story-mention triggers now resolve `wp_user_id` via `User_Manager`, enabling opt-out checks and consistent user tracking across all trigger types

## [2.10.0] — 2026-03-12
### Add
- **Reply message variants**: All automation slots now support multiple message variants; engine randomly picks one at execution time for more personalized, less monotone responses
- **Multi-step comment automation**: Comment auto-reply now supports an optional DM follow-up after the public comment reply, using any of the 10 reply types
- **DM follow-up panel**: Nested config panel in comment slot with full reply type selector, variants support, and all type-specific fields

## [2.9.1] — 2026-03-12
### Fix
- **Media picker**: Image/Video/Audio/File reply types now use WordPress Media Library selector instead of plain URL input
- **Payment Link fields**: Amount (required), Message (optional), Slug (optional — defaults to subsite subdomain); URL format `https://ig.pe/{slug}/{amount}`
- **Reply type re-render**: Reply type select change now dynamically updates fields without page refresh
- **Persistent Menu placeholder**: Value field placeholder dynamically switches between `https://...` and `PAYLOAD` based on selected type

## [2.9.0] — 2026-03-12
### Add
- **Rich reply types**: 10 reply options for automation slots — Text, Image, Video, Audio, File, Card (generic template), Buttons (button template), Media Share, Payment Link (ig.pe), Product Carousel (WooCommerce)
- **Product Carousel**: Dynamically builds Instagram generic template carousel from WooCommerce products with configurable Pay Now / Know More buttons
- **Payment Link**: ig.pe payment URL reply type for automation slots
- **Button list UI**: Add/remove button rows (max 3) for Card and Button template reply types, with web_url/postback type selector
- **Product picker**: Checkbox-based WooCommerce product selector for carousel configuration
- **Persistent Menu**: Account-level DM persistent menu management — add/remove/save menu items via Instagram messenger_profile API
- **Ice Breakers**: Account-level conversation starter questions — add/remove/save up to 4 questions via Instagram messenger_profile API
- **REST endpoints**: Products list, Persistent Menu CRUD, Ice Breakers CRUD under `/kdc/v1/instagram/`
- **API Client DELETE method**: `KDC_QTAP_IGCR_API_Client::delete()` for messenger_profile deletions
- **Messaging API methods**: `send_attachment()`, `send_button_template()`, `send_media_share()`, persistent menu and ice breaker management

## [2.8.1] — 2026-03-12
### Fix
- **Webhook persistence when Node is primary**: WP now always dispatches webhook entries locally for persistence (DM posts, comments, user creation, meta updates) regardless of `node_primary_webhook` setting — previously skipped entirely, causing messages/comments to not sync
- **Flow execution delegation**: Flow trigger handlers skip `Flow_Engine::dispatch()` when Node is primary, avoiding duplicate flow runs while preserving all data persistence

### Update
- **Node URL default**: `get_node_url()` defaults to `https://app.ig.cr` when no value is saved

## [2.8.0] — 2026-03-12
### Add
- **Flows block profile selector**: Account picker dropdown in automation settings header, allowing switching between Instagram profiles when multiple accounts are connected (mirrors Messages block pattern)

## [2.7.0] — 2026-03-12
### Refactor
- **Profile block redesign**: Clean minimal profile card with bordered container, compact layout
- **Action button bar**: Aligned pill buttons for Post, Message, Automate, Sync, Settings
- **Create Post modal**: Post creation moved from inline toggle to modal overlay, loaded non-intrusively
- **Content tabs**: Posts / Reels / Stories tabs with 3-column square grid, server-side rendered with JS tab switching
- **Single post actions**: Edit, Automate, Comment action buttons on single post view for admins
- **New icons**: Added grid, video, circle, globe, edit, message-square to SVG sprite
- **Admin panel collapsed**: Account meta, sync controls, disconnect zone hidden behind Settings button

---

## [2.6.0] — 2026-03-12
### Refactor
- **Automation flows rebuilt**: Replaced visual graph flow builder (SVG canvas, drag-drop nodes, Bezier edges, ~950 lines JS) with ManyChat-style settings panel using predefined automation slots
- **Slot-based engine**: 6 built-in slots (default_reply, welcome_message, story_mention_reply, comment_auto_reply, opt_in, opt_out) + unlimited user-created keyword rules
- **New `automation-slots.php`**: Slot registry, auto-creation, CRUD for `_igcr_flow_config` (flat JSON, version 3)
- **Simplified flow engine**: Flat `dispatch()` replaces recursive graph traversal (`execute_graph`, `traverse_from`)
- **DM dispatch priority chain**: opt_in/opt_out → keyword_rule (contains/equals/starts_with) → default_reply
- **REST endpoint**: `/kdc/v1/instagram/automation` (GET, PUT, PUT toggle, POST keyword, DELETE keyword)
- **Settings UI**: Collapsible cards with toggle switches, auto-save on input change, ~230 lines JS

### Remove
- Flow canvas UI (palette, SVG layer, node/port system, config panel, fullscreen mode)
- `flow-builder.js` (~950 lines), `flow-condition.php`, graph traversal engine
- `igcr_run_flow` cron action (graph-based wait steps no longer exist)
- ~650 lines of `.igcr-flow-*` CSS replaced with ~250 lines of `.igcr-auto-*` card/toggle styles

---

## [2.5.0] — 2026-03-12
### Add
- **Webhook cutover toggle**: Network Settings → "Primary Webhook Handler" checkbox makes Node.js the primary webhook processor
- **`is_node_primary_webhook()` getter**: Returns true only when Node is configured and the toggle is enabled
- When enabled, WP `receive()` skips `dispatch_entry()` — only verifies signature, logs, and relays to Node
- Node.js processes all webhook events via BullMQ (DM dispatch, comment handling, flow triggers)

---

## [2.4.0] — 2026-03-12
### Add
- **Node.js flow engine**: Full port of `Flow_Engine`, `Flow_Condition`, and `Flow_Action` from PHP to TypeScript
- **Graph traversal**: `flow-runner.ts` — recursive node traversal with condition branching, loop detection (max 100 visits), and wait step scheduling via BullMQ delayed jobs
- **Condition evaluator**: `flow-condition.ts` — 8 operators (contains, not_contains, equals, not_equals, starts_with, ends_with, is_empty, is_not_empty) with dot-notation field resolution
- **Action executor**: `flow-action.ts` — delegates send_dm, send_quick_reply, reply_comment, like_comment, add_tag, remove_tag to WordPress via REST API bridge
- **Flow resolver**: `flow-resolver.ts` — loads active flows + definitions directly from MySQL `wp_{blogId}_posts` + `wp_{blogId}_postmeta`
- **BullMQ flow-delay worker**: Processes delayed `wait` steps (replaces WP-Cron `igcr_run_flow`)
- **Flow trigger matching**: Webhook dispatcher now matches incoming events to automation flows after SSE publish
- **WP flow bridge endpoints**: `POST /kdc/v1/instagram/flow-action` and `POST /kdc/v1/instagram/flow-log` for Node → WP action execution and logging

---

## [2.3.0] — 2026-03-12
### Add
- **SSE client integration**: Frontend JavaScript connects to Node.js EventSource for real-time DM and comment push
- **Live DM updates**: Incoming messages append to open conversation panel in real time
- **Live comment updates**: New Instagram comments appear instantly in the comment thread without page reload
- **Toast notifications**: SSE notification events display as slide-in toasts
- **JWT token auto-refresh**: SSE client refreshes authentication token at 80% of TTL (before expiry)
- **SSE config localization**: `wp_localize_script()` passes `nodeUrl`, `blogId`, and `tokenUrl` to frontend
- **Data attributes**: `data-blog-id` on messages block and comments section for SSE channel targeting

---

## [2.2.0] — 2026-03-12
### Add
- **Webhook relay to Node.js**: WP webhook handler forwards payloads to Node via non-blocking `wp_remote_post()` (shadow mode — both process in parallel)
- **Node accepts WP relay**: Bearer token auth for relayed webhooks alongside direct Meta signature verification
- **Real-time SSE events**: Webhook dispatcher publishes `dm`, `comment`, and `notification` events to Redis pub/sub for live browser push
- **Structured event handling**: DM events (message, edit, postback, reaction, read) and field changes (comments, mentions, follow, media) mapped to typed SSE events

---

## [2.1.0] — 2026-03-12
### Add
- **Node.js microservice scaffold** (`igcr-node/`): Fastify + TypeScript sidecar for real-time webhooks, SSE push, and flow engine — Phase 1 foundation
- **Network Settings**: Node URL and shared secret fields for WP ↔ Node.js authentication
- **JWT realtime token endpoint** (`GET /kdc/v1/realtime/token`): issues short-lived tokens for SSE client authentication
- **Crypto port**: AES-256-CBC encryption in Node.js, byte-compatible with PHP `KDC_QTAP_IGCR_Crypto`
- **Webhook ingestion**: Node receives Meta webhooks, queues via BullMQ with deduplication and retry
- **SSE endpoint**: `GET /events/:blogId/:accountId` for real-time DM/comment push to browsers
- **PM2 + Nginx config**: production deployment config for Hostinger VPS (`app.ig.cr`)

---

## [2.0.0] — 2026-03-11
### Refactor
- **Network-wide accounts table**: Migrate `igcr_accounts` from per-site tables to a single network-wide table (`{base_prefix}igcr_accounts`) with `blog_id` column — eliminates O(n-sites) webhook lookup, makes all account operations context-independent
- `KDC_QTAP_IGCR_Instagram_Account` model now queries network table; `find_blog_id_by_ig_id()` reduced from N queries to 1
- `KDC_QTAP_IGCR_API_Client::for_account()` uses `base_prefix` — works from any blog context

### Fix
- **Flow engine cron bug**: `igcr_run_flow` (wait node resume) now carries `blog_id` in payload and switches to the correct subsite context before executing — previously ran in main-site context, causing subsite flows with wait nodes to silently fail
- **Media sync cron bug**: `igcr_media_sync_daily` now iterates all sites with active accounts — previously only synced accounts on the primary site
- **Media sync pagination bug**: `igcr_media_sync_page` now carries `blog_id` in cron args — subsequent pages fired in wrong context for subsites

### Update
- Token refresh (`igcr_token_refresh`) simplified to query network accounts table directly — no more `switch_to_blog()` loop across all sites
- Network Admin account actions (disconnect, delete, bulk ops) no longer require `switch_to_blog()` — row IDs are globally unique in the network table
- Account move (onboarding + network assign) now updates `blog_id` column instead of copy+delete between per-site tables
- `wp_delete_site` hook added to cascade-delete account rows when a subsite is removed
- Automatic migration on upgrade: copies all per-site `igcr_accounts` rows to network table with correct `blog_id`

## [1.11.1] — 2026-03-11
### Add
- Short URL redirect: `{site_url}/p/{slug}` resolves the subsite owning the Instagram post and 301-redirects to `{subsite}.{site_url}/{slug}/`

## [1.11.0] — 2026-03-11
### Fix
- Store Instagram Business Account ID (`ig_business_id` / `user_id` field) alongside the Instagram-Scoped User ID (`ig_user_id`) — webhooks use the IGBAID (`17841…`) in `entry.id` while Business Login OAuth returns the IGSID, causing all webhook-to-account lookups to fail silently
- OAuth callback now fetches `user_id` field from `/me` and stores it as `ig_business_id`
- `find_blog_id_by_ig_id()` and `get_by_ig_id()` now search both `ig_user_id` and `ig_business_id`

## [1.10.3] — 2026-03-11
### Fix
- Fix network Activity Log UNION query failing silently when some subsites lack `api_category` column — use explicit column list with fallback instead of `SELECT *`

## [1.10.2] — 2026-03-11
### Fix
- Fix keyword_match flow trigger reading keyword from wrong path in flow definition — was looking at `definition.trigger.keyword` instead of the trigger node's `parameters.keyword`

## [1.10.1] — 2026-03-11
### Fix
- Register flow trigger webhook hooks on bootstrap — `Flow_Engine::init()` was never called, so `igcr_webhook_dm` and all other webhook actions had no listeners, silently dropping all incoming webhook events and preventing flow automation

---

## [1.10.0] — 2026-03-11

### Add
- Unified design system with CSS custom properties (design tokens) across all plugin UI
- Network Admin accent color setting (Appearance section) — drives `--igcr-accent` across all blocks and views
- Lucide SVG icon sprite (`assets/icons/igcr-icons.svg`) with `igcr_icon()` PHP helper
- `.igcr-icon` base CSS class for consistent icon rendering

### Refactor
- Replace all hardcoded colors in `public.css` with CSS custom property references
- Replace HTML entity icons in flow palette with Lucide SVG icons
- Replace all inline SVGs in view templates with `igcr_icon()` calls
- Move flow node header colors from JS inline styles to CSS classes
- Standardize Activity Log inline colors to match design token palette
- Admin CSS: replace hardcoded colors with CSS custom properties

---

## [1.9.2] — 2026-03-11

### Add
- Flow editor: fullscreen toggle button with Escape key support

---

## [1.9.1] — 2026-03-11

### Add
- Activity Log: API Category filter (Media, Message, Comment, OAuth, Webhook, Account, Insights, Onboarding)
- Activity Log: API Category column with color-coded labels
- Auto-classification of log entries by endpoint pattern

---

## [1.9.0] — 2026-03-11

### Add
- Visual automation flow builder at `/flows/` — n8n-style node canvas with drag-and-drop, SVG Bezier connectors, and config panel
- Graph-based flow engine (v2) — replaces linear step engine with node+edges traversal supporting if/else branching
- REST API for flows: GET/POST/PUT/DELETE at `/kdc/v1/instagram/flows`
- `qtap/instagram-flows` Gutenberg block with auto-created `/flows/` page
- Flow list view with status cards, run counts, and activate/deactivate/delete actions
- Node palette with 5 triggers (DM Received, Keyword Match, Comment Received, Story Mention, New Follower), 2 logic nodes (Condition, Wait), and 6 action nodes (Send DM, Quick Reply, Reply Comment, Like Comment, Add/Remove Tag)
- Node config panel with parameter forms and variable hints

### Remove
- Legacy v1 linear flow engine (was never used in production)

---

## [1.8.3] — 2026-03-11

### Fix
- DM sync: use username-based participant matching (Instagram conversations API returns PSIDs, not Business Account IDs)
- DM sync: reduce conversation limit from 50 to 25 to avoid Instagram API error code 1
- DM sync: detect attachment types (photo, video, voice, etc.) for messages without text

---

## [1.8.2] — 2026-03-11

### Fix
- All qTap blocks now use `useBlockProps()` so they are clickable/selectable in the block editor canvas (not just via List View)

---

## [1.8.1] — 2026-03-11

### Add
- Sync DMs from Instagram API — fetches historical conversations and messages into local database
- Sync button in DM inbox header with animated spinner during sync
- Non-admin visitors see a "login as account owner" notice instead of blank page on /messages/

---

## [1.8.0] — 2026-03-11

### Add
- Dedicated `/messages/` frontend page with full DM inbox chat interface
- Two-panel layout: thread list + conversation view (side-by-side on desktop, stacked on mobile)
- REST API endpoints for DM management: `GET /kdc/v1/instagram/dm/threads`, `GET /kdc/v1/instagram/dm/messages`, `POST /kdc/v1/instagram/dm/send`
- Optimistic message sending with real-time UI updates
- Multi-account switcher in DM inbox header
- "Messages" link on Instagram profile page for quick access
- Outbound messages from automation flows now recorded in DM threads

---

## [1.7.0] — 2026-03-11

### Add
- Multi-type publishing from frontend profile page: Post (image), Reel (video), Album (carousel of 2–10 images), and Story
- Post type tab selector UI on the Create Post form with per-type file acceptance, dropzone icons, and validation
- Video preview with native controls for reel uploads
- Carousel thumbnail strip with per-image remove buttons for album posts
- Publish endpoint now accepts `media_type` (image/reel/carousel/story) and `media_ids` (array of attachment IDs)

---

## [1.6.0] — 2026-03-11

### Add
- Frontend "Create Post" interface on the Instagram profile page — users with `publish_posts` capability can upload an image, write a caption, and publish directly to Instagram without visiting wp-admin
- New REST endpoint `POST /kdc/v1/instagram/publish` — accepts an image attachment ID + caption, publishes to Instagram via Graph API, and creates a native WP post with all IGCR meta

### Remove
- Classic meta box for Instagram account selector on native posts — Gutenberg sidebar panel is now the sole UI

---

## [1.5.5] — 2026-03-11

### Fix
- Remove duplicate Instagram account selector — classic meta box now hidden when the block editor is active; only the Gutenberg sidebar panel shows
- Fix publish gate reverting posts to draft in Gutenberg — `wp_insert_post_data` now reads `_igcr_account_id` from the REST request body (`php://input`) since Gutenberg sends meta separately from `meta_input`

---

## [1.5.4] — 2026-03-11

### Fix
- Webhook comments now fetch username from Instagram API when missing — Instagram webhook payloads don't include `from.username`, so `GET /{comment_id}?fields=id,text,username,timestamp` is called to fill it
- Comment upsert now backfills empty `comment_author` on re-sync, so existing comments with missing usernames get fixed on next "Sync Comments"

---

## [1.5.3] — 2026-03-11

### Fix
- Post grid card: date now displays above caption
- Strip braille blank spacers (U+2800) from captions in grid cards and single post view — removes invisible padding Instagram users paste into captions

---

## [1.5.2] — 2026-03-11

### Fix
- Comments now display newest first (descending by timestamp)
- Show commenter username on all comments including the account's own replies

---

## [1.5.1] — 2026-03-11

### Add
- Full-screen sync overlay when "Sync Now" or "Sync Comments" buttons are clicked — shows spinner and "Syncing… please wait" message while the server processes the request

---

## [1.5.0] — 2026-03-11

### Add
- "Sync Comments" button on profile page — bulk-fetches comments from Instagram API for all synced posts of an account
- `sync_all_comments()` method in `KDC_QTAP_IGCR_Media_Sync` for programmatic bulk backfill

### Fix
- Single post template now uses `the_content` filter instead of `single_template` override — renders properly with both classic and block (FSE) themes
- `KDC_QTAP_IGCR_Comments::upsert()` now returns `int` (WP comment ID) instead of `void` — fixes silent failure of `_igcr_is_live` meta on live comments
- Theme's own `comments_template()` call no longer duplicates the IGCR comment section (suppressed via no-op template)

---

## [1.4.9] — 2026-03-08

### Add
- `qtap/instagram-posts` block auto-creates `/instagram-posts/` page on first load and sets it as static front page when no custom front page is configured

---

## [1.4.8] — 2026-03-08

### Add
- `qtap/igcr-post-grid` Gutenberg block: responsive Instagram post grid with configurable columns (2–6), post count, type filters (image/video/album/reel), and caption/date/badge toggles

### Fix
- `qtap/instagram-profile` ERR_TOO_MANY_REDIRECTS for non-admin users visiting `/profile/` — removed recursive subsite redirect loop
- `qtap/instagram-profile` "Access denied" for non-admins at `/profile/{username}/` — block now renders public content (header, bio, post grid) for all users; admin controls (sync bar, account meta, disconnect) shown only to admins

---

## [1.4.7] — 2026-03-08

### Fix
- **Onboarding clone: pages and footer not copying** — `wp_insert_post` / `wp_update_post` in multisite strips WordPress block comments (`<!-- wp:... -->`) via KSES for non-super-admin users, leaving all page and template-part content empty. Fixed by wrapping the entire write phase in `kses_remove_filters()` / `kses_init()`.
- **Onboarding clone: page meta not copied** — `_wp_page_template` and all other page meta are now copied to cloned pages.
- **Onboarding clone: incorrect page `orderby`** — changed `post_parent` to `parent` (the valid WP_Query orderby value) so parent pages are guaranteed to be inserted before their children.
- **Onboarding clone: block post lookup returns virtual theme-file templates** — added `suppress_filters => true` to both the source and lookup `get_posts` calls for FSE block posts to prevent WordPress from injecting file-based "virtual" template objects that have no real DB ID.

### Add
- **Onboarding clone: WooCommerce settings copied from template site** — currency, price formatting, tax settings, store address, email design options, and WC page assignments (shop/cart/checkout/my-account) are now cloned and WC page IDs remapped to the new site's page IDs.
- **Onboarding clone: domain replacement** — all template site URLs in page content, template-part content, custom CSS, and nav menu item URLs are replaced with the new site's domain during cloning.

---

## [1.4.6] — 2026-03-08

### Fix
- **`comment_received` / `live_comment_received` flow context**: Added missing `sender_id` (commenter's IG user ID from `$value['from']['id']`) so `send_dm` and `send_quick_reply` flow actions work correctly when triggered by an Instagram comment

---

## [1.4.5] — 2026-03-08

### Add
- **Full webhook field coverage**: All 18 Instagram webhook subscription fields are now dispatched, persisted, and wired to automation flow triggers:
  - **Messaging sub-types**: `messaging_postbacks` → `postback_received` trigger; `message_reactions` → `message_reaction` trigger; `messaging_referral` → `referral_received` trigger; `messaging_optins` → `optin_received` trigger
  - **Tracking-only** (no flow dispatch): `messaging_seen` updates `_igcr_last_seen_at` on the DM post; `message_edit` patches the edited message in `_igcr_messages`; `messaging_handover` logs the control-transfer event
  - **Change fields**: `live_comments` → `live_comment_received` trigger; `story_reactions` → `story_reaction` trigger; `follow` → `new_follower` trigger; `comment_poll_response` / `story_poll_response` → `poll_response` trigger; `share_to_story` → `share_to_story` trigger; `onboarding_welcome_message_series` logs only
- **`like_comment` flow action**: New automation step that likes an Instagram comment via `POST /{comment-id}/likes`
- **Activity Log messaging sub-types**: Webhook receive log now records specific field names (`messaging_postbacks`, `messaging_seen`, etc.) instead of the generic `messaging` label

### Fix
- **`shopping_product_tag_eligibility` API error**: Removed invalid field from `/me` Graph API request in the Instagram Profile block (400 on every profile render). Shop badge removed from view.
- **Full subscribed_fields**: OAuth callback and onboarding now subscribe to all 18 valid Instagram webhook fields

### Update
- Plugin version bumped `1.4.4` → `1.4.5`

---

## [1.4.4] — 2026-03-07

### Fix
- **Webhook subscription fix**: Removed invalid `feed` field from `subscribed_fields` in both OAuth callback and onboarding flows. The `subscribed_apps` API returned a 400 error (`Param subscribed_fields[0] must be one of {...} - got "feed"`). Subscription now registers only valid fields: `comments,messages,message_reactions,mentions,story_insights`

### Update
- Plugin version bumped `1.4.3` → `1.4.4`

---

## [1.4.3] — 2026-03-07

### Fix
- **Onboarding WooCommerce wizard**: New subsites no longer trigger the WooCommerce setup wizard on first admin visit. Sets `woocommerce_setup_wizard_complete`, marks `woocommerce_onboarding_profile` as completed+skipped, hides the task list, and deletes the `_wc_activation_redirect` transient — covers both the old WC setup wizard and the new React OBW (WC 7+)

### Update
- Plugin version bumped `1.4.2` → `1.4.3`

---

## [1.4.2] — 2026-03-07

### Fix
- **Onboarding template clone**: `clone_from_template()` now performs a full site clone. Previously only theme, options, and pages were copied. Now also copies:
  - Homepage settings (`show_on_front`, `page_on_front`, `page_for_posts`) with page IDs remapped to the new site
  - Nav menus — creates new menus, copies items with page ID remapping, fixes parent-child hierarchy, assigns to correct theme locations
  - Widgets — `sidebars_widgets` + all `widget_*` option arrays
  - Custom CSS (`wp_update_custom_css_post`)
  - `nav_menu_locations` stripped from `theme_mods` before bulk copy and rebuilt from new menu IDs to avoid stale term ID references
- **Site title**: New subsite title now uses the Instagram account display name (`$account->name`) instead of `'IGcr — slug'`

### Update
- Plugin version bumped `1.4.1` → `1.4.2`

---

## [1.4.1] — 2026-03-07

### Fix
- **Onboarding duplicate account**: `create()` now deletes the primary-site account row after successfully copying it to the new subsite. Previously the account existed on both the primary site and the new subsite simultaneously. The primary site is now correctly treated as an OAuth gateway only — the account's permanent home is the subsite.

### Update
- Plugin version bumped `1.4.0` → `1.4.1`

---

## [1.4.0] — 2026-03-07

### Add
- **Data Management at Uninstall** setting in Network Admin → App Settings
  - Three modes: **Keep all data** (default), **Delete all data**, **Selective**
  - Selective mode lets super admin choose individual categories: custom DB tables, plugin options, synced posts/DMs/flows, IG comments (`wp_comments` with `comment_type=igcr`), and `_igcr_*` post meta
  - Red warning banner reminding admin to take a backup before deleting
- `KDC_QTAP_IGCR_Network_Settings::get_uninstall_config()` — exposes mode + items for `uninstall.php`

### Update
- `uninstall.php` now reads the saved configuration and only removes the data categories the admin selected; defaults to keeping all data when no preference is saved
- Plugin version bumped `1.3.3` → `1.4.0`

---

## [1.3.3] — 2026-03-07

### Add
- **Admin Bar**: qTap IGcr menu under My Sites → Network Admin with links to App Settings, Plans, Site Assignments, Accounts, Activity Log, and Network Tools (visible to super admins only)
- `CLAUDE.md`: project-level standing instructions (auto-deploy, version bump, changelog on every change)

### Update
- Plugin version bumped `1.3.2` → `1.3.3`

---

## [1.3.2] — 2026-03-07

### Add
- Network Tools: **Migrate Legacy Comments** button — copies all rows from the old `{prefix}igcr_comments` custom table into native `wp_comments` for every site, matched to new synced posts via `_igcr_media_id`; idempotent (already-migrated comments are skipped via `_igcr_comment_id` dedup)

### Fix
- **Comments/mentions/DMs webhooks not received**: OAuth callback and onboarding `create()` now call `POST /{ig-user-id}/subscribed_apps?subscribed_fields=feed,comments,messages,message_reactions,mentions,story_insights` after connecting an account — Instagram requires this per-account call to forward webhook events; without it no events were delivered

### Update
- Plugin version bumped `1.3.1` → `1.3.2`

---

## [1.3.1] — 2026-03-07

### Add
- Network Settings: **Onboarding → Template Site** dropdown — select an existing subsite as a template; its theme, theme mods, pages (with parent hierarchy), and general WP options are cloned into every new subsite provisioned during self-serve onboarding
- `clone_from_template()` private method in `KDC_QTAP_IGCR_Onboarding` — copies active stylesheet/template, `theme_mods_{stylesheet}`, all published pages (old-ID → new-ID parent map), timezone/date/time/permalink options, and IGCR page-created markers so blocks do not recreate already-cloned pages
- `KDC_QTAP_IGCR_Network_Settings::get_template_blog_id()` getter

### Fix
- Network Accounts list: all accounts showed **Disconnected** — `get_all()` SELECT was missing `is_active`; added it so the column is returned and `(bool) $item['is_active']` evaluates correctly
- `_igcr_account_id` post meta now stores the permanent Instagram `ig_user_id` string (e.g. `"34416098348034898"`) instead of the local DB row `id` integer, so posts stay linked through account disconnect/reconnect cycles and username changes
- Profile block WP_Query updated to match against `ig_user_id` string with `'type' => 'CHAR'`; register_meta type changed from `integer` to `string`
- Gutenberg sidebar uses `a.ig_user_id` (string) as option value to avoid JavaScript integer-precision loss on 17-digit IDs
- Network Tools orphan detection and fix-account-ids handler updated to compare and store `ig_user_id` strings
- Single post template, comment endpoint, and media sync all updated to look up accounts via `get_by_ig_id()` instead of `get()`
- `admin/views/network/tools.php` parse error (duplicate `endforeach` fragment left by prior edit) that caused "Sorry, you are not allowed" on the Network Tools page

### Update
- Plugin version bumped `1.3.0` → `1.3.1`

---

## [1.3.0] — 2026-03-05

### Add
- Activity Log: native `WP_List_Table` — replaces custom table with WP-standard bulk delete, row-level delete, and WP pagination
- Activity Log details modal — `<dialog>` element shows Request/Response grid for API calls and full Payload for webhook events (no more inline expanding rows)
- Network Activity Log: **Site filter** dropdown — scopes aggregated log view to a single subsite
- Network Activity Log: **IG Account filter** dropdown — filters by `@username (Site Name)`, encoded as `blog_id:account_id`
- Network Setting: **Unique IG Accounts** checkbox — prevents the same Instagram account from being connected on more than one subsite; enforced at OAuth callback time with a user-visible error
- `KDC_QTAP_IGCR_Network_Settings::get_unique_accounts()` getter
- `KDC_QTAP_IGCR_Log_List_Table::set_network_accounts()` setter and `$network_accounts` property for network account filter
- `build_network_union_parts()` now accepts `blog_id` and `account_id` filter args for per-site and per-account scoping

### Update
- Network admin `render_activity_log_page()` — handles single and bulk delete with nonce verification; builds per-site account map for account filter
- OAuth `callback()` — enforces unique-accounts check before `switch_to_blog()` when the network setting is enabled
- Plugin version bumped `1.2.1` → `1.3.0`

### Fix
- Removed `string` type hint from `extra_tablenav($which)` override — parent `WP_List_Table` method has no type hint on older WordPress versions; PHP 8 strict declaration compatibility was throwing a fatal error
- Removed `string` type hint from `column_default($item, $column_name)` override — same root cause as above

---

## [1.2.0] — 2026-03-04

### Add
- `gridColumns` block attribute on `qtap/instagram-profile` — 3-col or 6-col post grid selectable from Block sidebar (Inspector Controls)
- Disconnect confirmation modal with aria/focus management and warning list covering posts, flows, token, and irreversibility
- `igcr-posts-grid--3col` / `igcr-posts-grid--6col` CSS modifiers; 6-col hides captions/dates (thumbnail-only grid)
- Network setting: **Graph API Version** field in App Settings — configurable `vNN.N` value (default `v25.0`) with format validation
- `KDC_QTAP_IGCR_API_Client::get_graph_api_url()` static method — reads version from network settings at runtime
- `KDC_QTAP_IGCR_Network_Settings::get_graph_api_version()` and `sanitize_api_version()` methods

### Update
- Profile block layout redesigned into grouped sections: Header (avatar + identity + stats), Bio + Website, Account Meta panel, Sync Bar, Post Grid, Danger Zone
- Sync bar now shows full WordPress date + time (`date_format` + `time_format`) instead of relative time
- Disconnect action replaced browser `confirm()` with accessible HTML modal (`role=dialog`, `aria-modal`, focus on cancel)
- Account meta migrated from `<dl>` to `.igcr-profile-meta` row-based panel with labels
- Block editor placeholder upgraded with `InspectorControls` RadioControl for 3/6-col grid selection
- `igcr-profile-wrap` max-width widened `560px` → `960px` to accommodate 6-col grid
- `blocks/instagram-profile/index.asset.php` — added `wp-block-editor`, `wp-components` to dependencies
- `KDC_QTAP_IGCR_API_Client::get()` and `post()` now call `self::get_graph_api_url()` (dynamic) instead of `self::GRAPH_API_URL` (hardcoded)
- Plugin version bumped `1.1.0` → `1.2.0`

---

## [1.1.0] — 2026-03-04

### Add
- `KDC_QTAP_IGCR_Media_Sync` class — imports Instagram posts into `igcr_post` CPT with pagination, rate limiting, and concurrency guard
- `igcr_media_sync_daily` WP-Cron event — daily background sync for all connected accounts
- `igcr_media_sync_page` single WP-Cron event — chains paginated imports (25 posts per run, 60 s gap)
- Webhook dispatch for `media` field changes → `igcr_webhook_media` action → auto-imports new posts on publish
- **Sync Now** button on `/profile/{username}/` — triggers immediate first-page sync + chains remaining pages
- Post grid on `/profile/{username}/` — card grid of synced posts (thumbnail, caption, date, type badge)
- "Last synced X ago" status line above post grid
- "Sync started" notice on redirect back after manual sync
- `IGCR_PLUGIN_BASENAME`, `IGCR_MIN_WP`, `IGCR_TEXT_DOMAIN` constants (were missing from explicit define list)
- `KDC_QTAP_IGCR_Activator::compat_notice()` static method for PHP version admin notice
- `KDC_QTAP_IGCR_API_Client` class constants: `GRAPH_API_URL`, `OAUTH_URL`, `TOKEN_URL`, `LONG_TOKEN_URL`, `REFRESH_TOKEN_URL`

### Update
- `KDC_QTAP_IGCR_API_Media::get_media()` — added `?string $after` cursor param for pagination; returns full response including `paging` object
- `/profile/{username}/` block view — added sync bar, post grid, and synced-notice sections
- `KDC_QTAP_IGCR_Block_Instagram_Profile::render()` — passes `$sync_now_url`, `$last_sync`, `$synced_notice` to profile view
- `KDC_QTAP_IGCR_Activator::activate()` — schedules `igcr_media_sync_daily` cron on activation
- `KDC_QTAP_IGCR_Deactivator::clear_scheduled_events()` — clears `igcr_media_sync_daily` and `igcr_media_sync_page` on deactivation
- `assets/css/public.css` — added `.igcr-sync-bar`, `.igcr-sync-now-link`, `.igcr-posts-grid`, `.igcr-post-card`, `.igcr-post-thumb-wrap`, `.igcr-post-caption`, `.igcr-post-time`, `.igcr-post-type-badge`, `.igcr-posts-empty`, `.igcr-sync-started-notice`
- Plugin version bumped `1.0.1` → `1.1.0`

### Refactor
- Moved Instagram API URL constants out of `kdc-qtap-igcr.php` into `KDC_QTAP_IGCR_API_Client` class constants; removed `IGCR_GRAPH_API_URL`, `IGCR_OAUTH_URL`, `IGCR_TOKEN_URL`, `IGCR_LONG_TOKEN_URL`, `IGCR_REFRESH_TOKEN_URL` global defines
- Moved PHP compat admin notice closure from `kdc-qtap-igcr.php` into `KDC_QTAP_IGCR_Activator::compat_notice()`
- Slimmed `kdc-qtap-igcr.php` to plugin header, core path/version constants, compat check, and three lifecycle hooks only

### Remove
- `IGCR_GRAPH_API_URL`, `IGCR_OAUTH_URL`, `IGCR_TOKEN_URL`, `IGCR_LONG_TOKEN_URL`, `IGCR_REFRESH_TOKEN_URL` global constants (replaced by `KDC_QTAP_IGCR_API_Client::*` class constants)

---

## [1.0.1] — 2026-03-03

### Add
- `/profile/{username}/` top-level page replacing `/accounts/edit/{username}/`
- `qtap/instagram-profile` block: inline stats row (Followers · Following · Posts) in profile header
- Shop badge (`shopping_product_tag_eligibility`) on profile
- Website link with globe icon on profile
- Account details section: Instagram ID, Page ID, Connected date, Token expires + "(will auto renew)"
- Reconnect button tooltip + `.igcr-reconnect-hint` subtitle
- `igcr_profile_rewrite_ver` migration (`'3'`) — moves existing `accounts/edit` page to top-level `profile` slug

### Update
- `qtap/instagram-accounts` block: renamed "Manage" button → "Profile Details"
- `KDC_QTAP_IGCR_API_Media::get_profile_data()` — added `website` and `shopping_product_tag_eligibility` fields
- `KDC_QTAP_IGCR_Block_Instagram_Profile::register()` — rewrite rule updated to `^profile/([^/]+)/?$`
- `blocks/instagram-profile/block.json` description updated

### Remove
- Profile edit form (biography / website POST) — Instagram Graph API does not support `POST /me` for profile updates
- `KDC_QTAP_IGCR_Block_Instagram_Profile::handle_update_profile()` method
- `admin_post_igcr_update_profile` hook

---

## [1.0.0] — Initial release

- Network-activated multisite plugin
- Instagram Business Login OAuth 2.0 flow
- Connected account management (`igcr_accounts` DB table)
- AES-256-CBC token encryption
- `qtap/instagram-accounts` Gutenberg block with auto-created `/accounts/` page
- `qtap/instagram-profile` Gutenberg block with auto-created `/profile/` page
- Custom Post Types: `igcr_post`, `igcr_reel`, `igcr_story`, `igcr_highlight`, `igcr_dm`, `igcr_flow`
- Automation flow engine with trigger / action / condition / log
- WP-Cron token refresh (`igcr_token_refresh`)
- WooCommerce product sync integration (HPOS-compatible)
- Webhook receiver with HMAC-SHA256 signature verification
- Frontend Instagram Login shortcode `[igcr_login_button]`
- Network admin plan management (Free / Starter / Pro)
