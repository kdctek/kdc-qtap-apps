# qTap IGcr — Feature Summary

**Version**: 2.51.0
**Plugin**: qTap IGcr (`kdc-qtap-igcr`)
**Platform**: WordPress Multisite (Network-activated)

---

## 1. Instagram Account Connection (OAuth 2.0)

qTap IGcr uses Instagram Business Login (the 2025 OAuth API, not the deprecated Basic Display API) to connect Instagram Business and Creator accounts. Each site can connect multiple accounts, with access tokens encrypted at rest using AES-256-CBC via OpenSSL. Upon successful OAuth, the plugin automatically calls `POST /{ig-user-id}/subscribed_apps` to subscribe to Instagram webhooks, ensuring real-time event delivery without manual configuration.

**Key capabilities:**

- OAuth 2.0 flow with state parameter format `{user_id}:{nonce}` for O(1) user resolution
- Multi-account support per site, each with independent token lifecycle
- Automatic webhook subscription on connect (`feed`, `comments`, `messages`, `message_reactions`, `mentions`, `story_insights`)
- Multisite-aware: `blog_id` passed through OAuth flow via transients for correct subsite routing
- Error display via transient-based redirect pattern with `igcr_error_key`

---

## 2. Media Sync

Instagram media is synced as native WordPress `post` type entries, differentiated by the `igcr_content_type` taxonomy with terms for each media format: `ig-image`, `ig-video`, `ig-album`, `ig-reel`, `ig-story`, and `ig-highlight`. Sync runs via a daily WP-Cron event and is also triggered in real-time by incoming webhook media events. The sync engine uses cursor-based pagination with concurrency locks to prevent duplicate processing across overlapping requests.

**Key capabilities:**

- Native `post` type storage with taxonomy-based content differentiation (no custom CPTs for media)
- Post meta keys: `_igcr_media_id`, `_igcr_media_url`, `_igcr_permalink`, `_igcr_caption`, `_igcr_timestamp`, `_igcr_is_synced`
- Cursor-based pagination for large media libraries
- Concurrency locks to prevent duplicate imports during overlapping sync runs
- Webhook-triggered incremental sync for new content

---

## 3. Webhook Event Processing

The plugin receives real-time events from Instagram via a verified webhook endpoint, processing comments, direct messages, mentions, story insights, message reactions, and follow events. Incoming payloads are verified using HMAC signature validation against the app secret. On multisite, each event is routed to the correct subsite by matching the Instagram user ID to a `blog_id` via `switch_to_blog()`, then dispatched as WordPress actions for downstream consumption.

**Key capabilities:**

- `GET /kdc/v1/instagram/webhook` for hub.challenge verification
- `POST /kdc/v1/instagram/webhook` for event ingestion
- HMAC-SHA256 signature verification on all incoming payloads
- WordPress action dispatch: `igcr_webhook_comment`, `igcr_webhook_dm`, `igcr_webhook_mention`, `igcr_webhook_media`
- Multisite routing: IG user ID resolved to `blog_id` via `find_blog_id_by_ig_id()`

---

## 4. Direct Message Management

Direct messages are stored as `igcr_dm` custom post type entries, with each post representing a conversation thread containing the full message history. Site administrators can view the DM inbox and reply to conversations directly from WordPress, with replies sent through the Instagram Graph API via `KDC_QTAP_IGCR_API_Messaging`. When a new DM sender is encountered, the plugin automatically creates a corresponding WordPress user account.

**Key capabilities:**

- `igcr_dm` CPT for conversation thread storage
- Full message history with chronological threading
- Reply via Instagram Graph API (`POST /{ig-user-id}/messages`)
- Automatic WordPress user creation for DM senders via `KDC_QTAP_IGCR_User_Manager`
- REST endpoint for inbox operations and message sending

---

## 5. Automation Flows

The automation system uses a slot-based architecture with predefined automation types: Default Reply (fallback DM when no keyword matches), Welcome Message (one-time DM to new followers), Story Mention Reply, Comment Auto-Reply, and user-created Keyword Rules. Each slot is backed by an `igcr_flow` CPT post with its configuration stored as JSON in post meta. The flow engine supports keyword matching, chained actions, version history, and WP-Cron scheduling for delayed execution of wait steps.

**Key capabilities:**

- 5 built-in slot types: default reply, welcome message, story mention reply, comment auto-reply, keyword rules
- Keyword matching with configurable rules and chained actions
- Flow definitions stored as versioned JSON in `_igcr_flow_definition` post meta
- WP-Cron integration for `wait` step scheduling via `igcr_run_flow` events
- Plan-gated: flow count limits enforced via `KDC_QTAP_IGCR_Site_Plan::is_allowed()`

---

## 6. Comment Management

Instagram comments are stored in the native WordPress `wp_comments` table with `comment_type = 'igcr'`, leveraging WordPress's built-in comment infrastructure for threading, moderation, and display. Parent-child threading is maintained by resolving Instagram's `ig_parent_id` to the corresponding WordPress `comment_ID` via comment meta lookup. The `upsert()` method ensures idempotent processing, preventing duplicate comments from repeated webhook deliveries.

**Key capabilities:**

- Native `wp_comments` storage with `comment_type = 'igcr'`
- Deduplication via `comment_meta: _igcr_comment_id` (Instagram comment ID)
- Parent threading: `ig_parent_id` resolved to WordPress `comment_ID`
- Reply-to-comment via Instagram Graph API (`POST /kdc/v1/comments/{id}/reply`)
- Legacy migration tool: copies from old `{prefix}igcr_comments` table to `wp_comments`

---

## 7. Content Publishing

WordPress posts can be published directly to Instagram as images, carousels, reels, or stories through the Instagram Graph API content publishing endpoints. A Gutenberg sidebar panel (`post-ig-sidebar.js`) allows authors to select the target Instagram account before publishing. The plugin validates media format and size requirements before submission and reverts posts to draft if no account is selected, preventing accidental publishes without Instagram targeting.

**Key capabilities:**

- Publish to Instagram: image, carousel, reel, and story formats
- Gutenberg sidebar panel for Instagram account selection (no build step, uses `wp.element.createElement`)
- Draft validation gate: posts without account selection reverted to draft on save
- Media format and size pre-checks before API submission
- Post meta flag `_igcr_is_synced` distinguishes synced-from-IG posts from publish-to-IG posts

---

## 8. WooCommerce Integration

qTap IGcr integrates with WooCommerce in an HPOS-compatible manner, adding Instagram product sync fields to WooCommerce products for catalog integration. Feature availability is gated by the site's assigned plan limits, allowing network administrators to control which sites have access to commerce features. Users auto-created from Instagram DM interactions are assigned the WooCommerce `customer` role for seamless storefront access.

**Key capabilities:**

- High-Performance Order Storage (HPOS) compatible
- Product sync fields for Instagram catalog integration
- Plan-gated access: product sync limits enforced per site
- Auto-created DM users assigned `customer` role for WooCommerce storefront
- WC version requirements: 7.0+ (tested up to 10.5.3)

---

## 9. Multi-Site Plan Management

A network-level plan system allows super administrators to define plans with configurable limits for connected accounts, synced posts, WooCommerce products, and automation flows. Plans are stored in the `wp_igcr_plans` network table and assigned to individual subsites via `wp_igcr_site_plans`. All limit checks are centralized through `KDC_QTAP_IGCR_Site_Plan::is_allowed()`, with a built-in free plan fallback ensuring baseline functionality for unassigned sites.

**Key capabilities:**

- Network-wide plan definitions with configurable resource limits
- Per-site plan assignment via `wp_igcr_site_plans` table
- Centralized enforcement via `is_allowed()` method
- Free plan fallback for sites without explicit assignment
- CRUD interface in Network Admin for plan management

---

## 10. Gutenberg Blocks

The plugin provides 7 Gutenberg blocks for frontend rendering: Accounts listing, Instagram Profile (grid with tabs), Post Grid (with filtering and pagination), DM Inbox, Flows Builder, Onboarding wizard, and Navigation Menu. Blocks are registered server-side with PHP render callbacks and do not require a JavaScript build step. On plugin activation, corresponding pages are auto-created with the appropriate block markup to provide an immediate working frontend.

**Key capabilities:**

- 7 blocks: Accounts, Instagram Profile, Post Grid, DM Inbox, Flows, Onboarding, Navigation Menu
- Server-side rendering via PHP callbacks (no React build step)
- Auto-created pages on activation for immediate frontend setup
- Instagram Profile block: grid layout with tab-based content navigation
- Post Grid block: configurable filtering by content type with pagination

---

## 11. Instagram Login

Frontend users can log in to WordPress using their Instagram identity via a dedicated OAuth flow. The `[igcr_login_button]` shortcode renders a login button that can be placed on any page, while hooks on `wp-login.php` and WooCommerce login forms provide native integration points. On authentication, the plugin creates a new WordPress user or matches an existing one by Instagram identity, enabling frictionless account creation.

**Key capabilities:**

- `[igcr_login_button]` shortcode for placement on any page or widget
- Hooks on `wp-login.php` and WooCommerce login/registration forms
- Automatic user creation or matching by Instagram identity
- OAuth-based authentication (same Instagram Business Login API)
- Managed by `KDC_QTAP_IGCR_Instagram_Login` class

---

## 12. Activity Logging

Every significant plugin operation is recorded in a per-site activity log, including API calls, webhook events, sync operations, and administrative actions. Logs are automatically pruned after 90 days to manage database growth. A `WP_List_Table`-based admin viewer provides sortable, paginated access to log entries at both the site and network level.

**Key capabilities:**

- Per-site and network-wide activity logging
- Automatic 90-day log pruning via scheduled cleanup
- `WP_List_Table`-based admin interface with sorting and pagination
- Logs API calls, webhook deliveries, sync operations, and admin actions
- Managed by `KDC_QTAP_IGCR_Activity_Log` and `KDC_QTAP_IGCR_Log_List_Table`

---

## 13. Site Cloning / Onboarding

The onboarding system provides a self-serve flow for new users: connect an Instagram account, choose a subdomain, and have a fully configured subsite created automatically. New sites can be cloned from a template site, copying over accounts, automation flows, and settings to provide a pre-configured starting point. The entire onboarding process is driven through a REST API with a corresponding Gutenberg block for the frontend wizard interface.

**Key capabilities:**

- Self-serve onboarding: connect Instagram, choose subdomain, create subsite
- Template site cloning via `KDC_QTAP_IGCR_Site_Cloner` (accounts, flows, settings)
- REST API-driven onboarding flow (`KDC_QTAP_IGCR_Onboarding`)
- Gutenberg Onboarding block for frontend wizard UI
- Automatic plan assignment for newly created sites

---

## 14. Token Security

All Instagram access tokens are encrypted at rest using AES-256-CBC via PHP's OpenSSL extension, with the encryption key derived from the WordPress `AUTH_KEY` constant. A daily WP-Cron event (`igcr_token_refresh`) automatically refreshes tokens before they expire, maintaining the 60-day token lifespan mandated by the Instagram Graph API. The encryption layer includes graceful fallback behavior when OpenSSL is unavailable, ensuring the plugin remains functional in constrained hosting environments.

**Key capabilities:**

- AES-256-CBC encryption via OpenSSL for all stored tokens
- Encryption key derived from WordPress `AUTH_KEY` constant
- Daily automatic token refresh via `igcr_token_refresh` cron event
- 60-day token lifespan management with proactive renewal
- Graceful degradation when OpenSSL extension is unavailable

---

## 15. Network Administration

The network admin panel provides 6 sections for centralized multisite management: App Settings (organized into 4 tabs: App, Accounts, Appearance, Advanced), Plans CRUD, Site Assignments, Connected Accounts list, Activity Log, and Network Tools. The Tools section offers maintenance utilities including Flush Sync Cursors, Migrate Legacy Comments, and Fix Orphaned Post Account IDs. An admin bar entry under "My Sites > Network Admin" gives super administrators quick access.

**Key capabilities:**

- 6-section admin panel: App Settings, Plans, Site Assignments, Accounts, Activity Log, Tools
- App Settings split into 4 tabs: App, Accounts, Appearance, Advanced (each saves independently)
- Network Tools: flush sync cursors, migrate legacy comments, fix orphaned post account IDs
- Connected Accounts list table across all subsites (`KDC_QTAP_IGCR_Accounts_List_Table`)
- Admin Bar quick-access for super administrators

---

## 16. Design System

The plugin ships with a configurable design system anchored by a brand accent color (default `#3D6792`) that can be changed in Network Settings. CSS custom properties are defined on `:root` and dynamically overridden via `wp_add_inline_style()` from the `get_accent_css_vars()` method. An icon system based on Lucide SVG sprites (`assets/icons/igcr-icons.svg`) is available through the `igcr_icon()` PHP helper, providing consistent iconography across all admin and frontend views.

**Key capabilities:**

- Configurable accent color via Network Settings (Appearance tab)
- CSS custom properties on `:root` with dynamic overrides via inline styles
- Lucide SVG icon sprite with `igcr_icon( $name, $size )` PHP helper
- Responsive layouts across admin and frontend views
- Consistent design language across blocks, admin pages, and frontend templates

---

## 17. Data Deletion Compliance

The plugin implements Meta's required data deletion callback at `POST /kdc/v1/instagram/data-deletion`, verifying incoming requests using HMAC-SHA256 against the app secret. Administrators can configure one of 3 deletion modes in Network Settings: delete data (permanently remove account, tokens, and user meta), deactivate only (soft-delete while preserving data), or do nothing (log only). Each deletion event generates a unique confirmation code with a status check endpoint that Meta can poll, and triggers both a non-dismissible admin notice and email notification to site and network administrators.

**Key capabilities:**

- HMAC-SHA256 signed request verification against app secret
- 3 configurable modes: delete data, deactivate only, do nothing (log only)
- Confirmation code generation with `GET /kdc/v1/instagram/data-deletion/status` polling endpoint
- Non-dismissible admin notice (requires manual acknowledgement) on deletion events
- Email notifications to site admin and network admin with full event details

---

## 18. Real-Time Updates

An optional Node.js microservice provides real-time event delivery to the frontend via Server-Sent Events (SSE). WordPress issues short-lived JWT tokens (15-minute expiry) through the `GET /kdc/v1/realtime/token` REST endpoint, signed with a shared secret between WordPress and the Node.js service. The Node.js layer uses BullMQ as a job queue to relay webhook events from WordPress to connected SSE clients, enabling live updates for DM inboxes and comment streams without polling.

**Key capabilities:**

- Server-Sent Events (SSE) for real-time frontend updates
- JWT-authenticated tokens issued by WordPress (15-minute expiry via `KDC_QTAP_IGCR_Realtime_Endpoint`)
- Shared secret (`node_secret`) between WordPress and Node.js for token signing
- BullMQ job queue for reliable webhook-to-SSE event relay
- SSE endpoint: `GET /events/:blogId/:accountId?token=<jwt>`

---

## Technical Stack

| Component | Details |
|---|---|
| **PHP** | 8.0+ (uses match expressions, named arguments, union types, nullsafe operator) |
| **WordPress** | 6.0+ (tested up to 6.9.2), Network-activated Multisite |
| **WooCommerce** | 7.0+ (tested up to 10.5.3), HPOS-compatible |
| **Instagram API** | Graph API v25.0 via Instagram Business Login (2025) |
| **Encryption** | AES-256-CBC via OpenSSL, key derived from `AUTH_KEY` |
| **Real-Time** | Node.js + BullMQ + SSE (optional microservice) |
| **Authentication** | JWT (real-time tokens), HMAC-SHA256 (webhooks, data deletion) |
| **Coding Standards** | WordPress Coding Standards (WPCS) compliant |
| **Frontend** | No jQuery (except admin.js), no React build step, Lucide SVG icons |
| **Database** | Network tables (`wp_igcr_plans`, `wp_igcr_site_plans`), per-site tables (`igcr_accounts`, `igcr_flow_logs`) |
