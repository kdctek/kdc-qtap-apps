# qTap IGcr — Support Knowledge Base

> Structured Q&A reference for the qTap IGcr WordPress plugin support chatbot.
> Plugin version: 2.50.0 | WordPress Multisite (network-activated) | PHP 8.0+

---

## Super Admin (Network Administrator)

### Getting Started

**Q: How do I configure the Meta App credentials?**

A: Navigate to **Network Admin > qTap IGcr > App Settings**. Switch to the **App** tab. Enter your **Meta App ID** and **App Secret** in the corresponding fields. Click **Save Changes**. The plugin will encrypt the App Secret using AES-256-CBC before storing it. Until these credentials are saved, no Instagram accounts can be connected on any subsite.

---

**Q: Where do I find my Meta App ID and App Secret?**

A: Go to [developers.facebook.com](https://developers.facebook.com), open your app, and navigate to **App Settings > Basic**. Your App ID is shown at the top. The App Secret is hidden by default -- click "Show" and enter your Facebook password to reveal it. Copy both values into the plugin's App Settings page.

---

**Q: How do I set up the webhook verify token?**

A: The plugin auto-generates a unique 32-character verify token when you first save App Settings. You do not need to create one manually. To view the generated token, go to **Network Admin > qTap IGcr > App Settings** (App tab). The "Webhook Verify Token" field displays the auto-generated value. Copy this token and paste it into your Meta App's webhook configuration under **Webhooks > Instagram** in the Meta Developer Dashboard.

---

**Q: What webhook URL do I give to Meta?**

A: Your webhook URL follows this format:

```
https://your-primary-site.com/wp-json/kdc/v1/instagram/webhook
```

This is the REST endpoint registered on your primary (main) site. In the Meta Developer Dashboard, go to **Webhooks > Instagram**, click "Edit Subscription", and enter:
- **Callback URL**: The URL above
- **Verify Token**: The auto-generated token from App Settings

Subscribe to the following fields: `messages`, `comments`, `mentions`, `story_insights`, `feed`, `message_reactions`.

The plugin handles both the GET verification challenge and POST event payloads at this single URL.

---

### Plans & Billing

**Q: How do I create a new plan?**

A: Navigate to **Network Admin > qTap IGcr > Plans**. Click the **Add New Plan** button. Fill in the plan details:
- **Name**: Display name (e.g., "Pro", "Business")
- **Slug**: URL-safe identifier (auto-generated from name if left blank)
- **Description**: Optional plan description
- **Max Accounts**: Number of Instagram accounts allowed (-1 for unlimited)
- **Max Posts**: Monthly post limit (-1 for unlimited)
- **Max Products**: WooCommerce product sync limit (-1 for unlimited)
- **Max Flows**: Number of automation flows allowed (-1 for unlimited)
- **Price**: Plan price
- **Period**: Billing period (monthly or annual)

Click **Create Plan** to save.

---

**Q: What limits can I configure on a plan?**

A: Plans support four configurable limits:
1. **Max Accounts** (`max_accounts`) -- Number of Instagram accounts a site can connect. Default: 1.
2. **Max Posts** (`max_posts`) -- Monthly post publishing limit. Default: unlimited (-1).
3. **Max Products** (`max_products`) -- Number of WooCommerce products that can be synced. Default: unlimited (-1).
4. **Max Flows** (`max_flows`) -- Number of automation flows a site can create. Default: 5.

A value of **-1** means unlimited for any limit. Plans also have a price and billing period (monthly or annual).

---

**Q: How do I assign a plan to a subsite?**

A: Navigate to **Network Admin > qTap IGcr > Site Assignments**. You will see a list of all subsites in the network. For each site, select a plan from the dropdown and click **Assign**. Each site can have one active plan at a time. Assigning a new plan replaces the previous one.

---

**Q: What happens when a site exceeds its plan limits?**

A: When a site reaches a plan limit:
- **Accounts**: The "Connect Instagram Account" button will show an error: "Account limit reached for your current plan. Please upgrade to connect more accounts." The OAuth flow is blocked before reaching Instagram.
- **Flows**: Creating new automation flows is blocked with a plan limit message.
- **Posts**: Publishing to Instagram is blocked when the monthly limit is reached.
- **Products**: WooCommerce product sync is limited to the plan's max_products count.

Existing connected accounts and data are never deleted when a plan limit is reached -- only new additions are blocked.

---

**Q: What are the default (free) plan limits?**

A: When no plan is assigned to a site, it operates on the implicit free tier. The free tier behavior depends on what `KDC_QTAP_IGCR_Site_Plan::is_allowed()` returns when no plan exists -- by default, it allows the action (returns true), so sites without a plan have no enforced limits. To restrict unassigned sites, create a "Free" plan with your desired limits and assign it to all sites.

---

### Network Tools

**Q: How do I flush sync cursors?**

A: Navigate to **Network Admin > qTap IGcr > Network Tools**. Find the **Flush Sync Cursors** section and click the button. This clears all `igcr_sync_cursor_{id}` options across every subsite in the network. Use this when media sync is stuck mid-pagination or when you want to force a full re-sync from page one on the next cron run. Flushing cursors does not delete any synced posts -- it only resets the pagination state.

---

**Q: How do I migrate legacy comments?**

A: Navigate to **Network Admin > qTap IGcr > Network Tools**. Find the **Migrate Legacy Comments** section and click the button. This copies comment rows from the old `{prefix}igcr_comments` custom table into native WordPress `wp_comments` entries with `comment_type = 'igcr'`. Comments are matched to posts via `_igcr_media_id` post meta. The legacy table is not deleted -- it remains as a backup. Migration is idempotent: running it twice will not create duplicate comments because deduplication uses the `_igcr_comment_id` meta key.

---

**Q: How do I fix orphaned post account IDs?**

A: Navigate to **Network Admin > qTap IGcr > Network Tools**. Find the **Fix Orphaned Post Account IDs** section and click the button. This scans all synced posts across the network and checks whether their `_igcr_account_id` post meta points to a valid, active account in the `igcr_accounts` table. If the referenced account no longer exists (e.g., it was disconnected or deleted), the tool clears the orphaned meta so the post is no longer associated with a nonexistent account.

---

### Accounts Management

**Q: How do I view all connected accounts across the network?**

A: Navigate to **Network Admin > qTap IGcr > Accounts**. This page displays a list table showing every connected Instagram account across all subsites. Each row shows the username, account type (Business/Creator), connected site (blog_id), connection date, and token expiry date. You can sort and filter the list. The table uses the `KDC_QTAP_IGCR_Accounts_List_Table` class (extending `WP_List_Table`).

---

**Q: How do I disconnect an account from the network level?**

A: Navigate to **Network Admin > qTap IGcr > Accounts**. Find the account you want to disconnect in the list. Hover over the row to reveal the **Disconnect** action link. Click it and confirm the action. This sets `is_active = 0` on the account row. The account's access token is preserved (encrypted) so it can be reconnected later without re-authorizing. To permanently delete an account and its token, use the **Delete** action instead.

---

### Activity Log

**Q: How do I view the network-wide activity log?**

A: Navigate to **Network Admin > qTap IGcr > Activity Log**. This page shows a chronological log of all plugin events across the network, including OAuth connections, webhook events, API calls, sync operations, and errors. Each log entry shows the event type, direction (inbound/outbound/system), endpoint, HTTP status, timestamp, and a summary. You can filter by event type and error status.

---

**Q: How long are logs retained?**

A: Log retention is configured in **Network Admin > qTap IGcr > App Settings** under the **Advanced** tab. The "Log Retention" setting accepts these values:
- **0** (default): Logs are never auto-deleted.
- **7**: Logs older than 7 days are purged.
- **30**: Logs older than 30 days are purged.
- **90**: Logs older than 90 days are purged.
- **365**: Logs older than 1 year are purged.

Purging happens automatically. Set this to a non-zero value if your database is growing too large from log entries.

---

### Data Deletion

**Q: How do I configure data deletion behaviour?**

A: Navigate to **Network Admin > qTap IGcr > App Settings** (App tab). Find the **Deletion Mode** setting. Select one of the three modes and click **Save Changes**. This controls what happens when a user removes your app from their Instagram account (Instagram sends a data deletion callback to your endpoint at `/wp-json/kdc/v1/instagram/data-deletion`).

---

**Q: What are the 3 deletion modes?**

A: The three deletion modes are:

1. **Delete** (`delete`) -- Default. Permanently removes the account row from the `igcr_accounts` table and clears all `_igcr_ig_user_id`, `_igcr_ig_username`, and `_igcr_ig_name` user meta from linked WordPress users. Synced posts and comments are NOT deleted (they are considered published site content). An email notification is sent to the site admin and network admin.

2. **Deactivate** (`deactivate`) -- Sets the account to `is_active = 0` (disconnects it) but preserves all data. This lets you review the account before deciding whether to delete it manually. An email notification is sent.

3. **Ignore** (`ignore`) -- Logs the deletion request but takes no action on data. No email is sent. Use this only if you have a manual process for handling deletion requests. Note: Meta requires apps to honor data deletion requests, so using "ignore" permanently may violate Meta's platform policies.

---

**Q: How do I acknowledge a data deletion notice?**

A: When a data deletion request is received (in "delete" or "deactivate" mode), a persistent orange admin notice appears at the top of every admin page. The notice shows the Instagram username, the date, the action taken, and a confirmation code. Click the **Acknowledge & dismiss** button to remove the notice. This does not undo the deletion -- it simply clears the notice. The notice persists across page loads until acknowledged because it is stored in a network site option (`igcr_data_deletion_notices`).

---

### Settings

**Q: How do I change the accent color?**

A: Navigate to **Network Admin > qTap IGcr > App Settings** and switch to the **Appearance** tab. Use the **Accent Color** color picker to select a new hex color. The default is `#0095F6` (Instagram's primary blue). Click **Save Changes**. The accent color drives all CSS custom properties across the plugin's frontend:
- `--igcr-accent` (primary buttons, links)
- `--igcr-accent-hover` (hover states, 12% darker)
- `--igcr-accent-active` (pressed states, 24% darker)
- `--igcr-accent-soft` (badges, 15% opacity)
- `--igcr-accent-light` (backgrounds, 8% opacity)
- `--igcr-accent-border` (borders, 25% opacity)

---

**Q: How do I configure the Graph API version?**

A: Navigate to **Network Admin > qTap IGcr > App Settings** (App tab). Find the **Graph API Version** field. Enter the version string in the format `vNN.N` (e.g., `v25.0`, `v26.0`). The default is `v25.0`. Click **Save Changes**. All Instagram Graph API requests will use this version. Only update this when Meta deprecates the current version and you have tested with the new one.

---

## Admin (Site Administrator)

### Connecting Accounts

**Q: How do I connect an Instagram account?**

A: Navigate to **qTap IGcr > Accounts** in your site's admin menu. Click the **Connect Instagram Account** button. You will be redirected to Instagram's authorization page where you log in and approve the permissions. After approval, Instagram redirects back to your site and the account appears in your accounts list. The plugin:
1. Exchanges the authorization code for a short-lived token
2. Exchanges the short-lived token for a long-lived token (valid 60 days)
3. Encrypts and stores the token
4. Subscribes to webhook events for this account

---

**Q: Why does my account connection fail with "redirect_uri mismatch"?**

A: This error occurs when the redirect URI sent to Instagram during OAuth does not exactly match what is configured in your Meta App. Common causes:
1. **HTTP vs HTTPS**: Your site URL uses `http://` but the Meta App has `https://` (or vice versa).
2. **www vs non-www**: Your site uses `www.example.com` but the Meta App has `example.com`.
3. **Trailing slash differences**: One URL has a trailing slash, the other does not.

To fix this:
1. Go to your Meta App Dashboard > Instagram Business Login > Settings.
2. Check the **Valid OAuth Redirect URIs** list.
3. Add the exact callback URL: `https://your-primary-site.com/wp-json/kdc/v1/instagram/oauth/callback`
4. Make sure the protocol (http/https) and domain match your WordPress site URL exactly.

The plugin stores the redirect_uri in a transient during `initiate()` and reuses the exact same value in `callback()` to prevent byte-for-byte mismatches.

---

**Q: How many accounts can I connect?**

A: The number of accounts depends on your site's assigned plan. Each plan has a `max_accounts` limit. If no plan is assigned, there is no enforced limit. You can check your limit by asking your Network Administrator which plan is assigned to your site. When the limit is reached, clicking "Connect Instagram Account" returns the error: "Account limit reached for your current plan."

---

**Q: How do I disconnect an account?**

A: Navigate to **qTap IGcr > Accounts**. Find the account you want to disconnect. Click the **Disconnect** action link next to the account. This sets the account to inactive (`is_active = 0`) but preserves the encrypted token and data. The account can be reconnected later by clicking "Connect Instagram Account" and authorizing the same Instagram account -- the plugin will reactivate the existing row via upsert.

---

**Q: What permissions does the plugin request from Instagram?**

A: The plugin requests four Instagram Business Login scopes:
1. **instagram_business_basic** -- Read profile info, media, and account data.
2. **instagram_business_manage_messages** -- Send and receive direct messages.
3. **instagram_business_manage_comments** -- Read and reply to comments.
4. **instagram_business_content_publish** -- Publish photos, videos, reels, and carousels to the feed.

These scopes use the 2025 Instagram Business Login API (not the deprecated Basic Display API).

---

### Media Sync

**Q: How does media sync work?**

A: Media sync imports Instagram posts into WordPress as native `igcr_media` CPT entries. The process:
1. Calls the Instagram Graph API `/{ig-user-id}/media` endpoint with pagination (25 items per page).
2. For each media item, creates or updates a WordPress post with the caption as content, the first sentence as the title, and the media URL as a featured image (sideloaded locally).
3. Stores Instagram metadata in post meta (`_igcr_media_id`, `_igcr_media_url`, `_igcr_permalink`, `_igcr_timestamp`, etc.).
4. Sets the `_igcr_is_synced = 1` flag to mark it as a synced post.
5. Assigns an `igcr_content_type` taxonomy term based on media type: `ig-image`, `ig-video`, `ig-album`, `ig-reel`, `ig-story`, or `ig-highlight`.
6. For carousel/album posts, fetches child items and stores them in `_igcr_album_children` as a JSON array.

---

**Q: How often does media sync run?**

A: Media sync runs on two schedules:
1. **Daily cron** (`igcr_media_sync_daily`): Syncs one page (25 items) for each active account across the network. If more pages exist, it schedules a single follow-up event (`igcr_media_sync_page`) to fetch the next page.
2. **Webhook-triggered**: When Instagram sends a `feed` webhook event (new post published), the plugin immediately imports that single post without waiting for the daily cron.

A concurrency guard (transient `igcr_sync_running_{id}` with a 5-minute TTL) prevents overlapping sync runs for the same account. Pages are fetched at least 60 seconds apart to stay under the API rate limit (200 calls/hour).

---

**Q: How do I trigger a manual sync?**

A: Navigate to **qTap IGcr > Accounts** in your site admin. Find the account you want to sync and click the **Sync Now** button. This triggers `igcr_sync_media_now` via `admin-post.php`, which starts the sync immediately for that account. You will be redirected back to the accounts page when done. If a sync is already running (concurrency guard active), the manual sync will be skipped.

---

**Q: Why are some posts missing from the sync?**

A: Common reasons for missing posts:
1. **Pagination incomplete**: The initial sync fetches 25 posts per cron run. If you have hundreds of posts, it may take several cron cycles to import them all. Check if a sync cursor exists (`igcr_sync_cursor_{id}` option) -- if it does, the sync is still in progress.
2. **Stories expire**: Instagram Stories are only available via the API for 24 hours. If the sync did not run within that window, the story will not be imported.
3. **API rate limits**: If the plugin exceeds 200 API calls/hour, subsequent requests fail silently and are retried on the next cron run.
4. **Reels vs. Posts**: Ensure you are checking the correct content type filter. Reels are tagged as `ig-reel` in the `igcr_content_type` taxonomy.
5. **Stuck sync**: If `igcr_sync_running_{id}` transient is stuck, use **Network Admin > qTap IGcr > Network Tools > Flush Sync Cursors** to reset.

---

**Q: What media types are supported?**

A: The plugin supports all Instagram media types:
- **IMAGE** -- Single photo posts (taxonomy: `ig-image`)
- **VIDEO** -- Video posts (taxonomy: `ig-video`)
- **CAROUSEL_ALBUM** -- Multi-image/video carousel posts (taxonomy: `ig-album`). Child items are stored in `_igcr_album_children` post meta.
- **REEL** -- Instagram Reels (taxonomy: `ig-reel`). Thumbnail URL is used for the featured image since reels use `thumbnail_url` instead of `media_url` for the poster frame.
- **STORY** -- Instagram Stories (taxonomy: `ig-story`). Only available for 24 hours via the API.
- **HIGHLIGHT** -- Story Highlights (taxonomy: `ig-highlight`)

---

### Automation Flows

**Q: How do I create a keyword auto-reply?**

A: Navigate to **qTap IGcr > Accounts** and select the account you want to automate. Go to the **Automation** section. Keyword rules are managed as automation slots. To create a keyword rule:
1. Click **Add Keyword Rule** (or the equivalent button in the automation panel).
2. Enter the **keyword(s)** to match against incoming DMs.
3. Configure the **reply type** (text message, quick reply, attachment, template, etc.).
4. Enter the **reply message** content. You can use placeholder variables for personalization.
5. Save the rule.

When someone sends a DM containing the keyword, the flow trigger checks keyword rules in priority order: opt_in/opt_out first, then keyword_rules, then the default_reply fallback.

---

**Q: What are automation slots?**

A: Automation slots are predefined automation types, each backed by an `igcr_flow` CPT post. The built-in slot types are:
1. **Default Reply** -- Fallback DM reply when no keyword rule matches an incoming message.
2. **Welcome Message** -- One-time DM sent automatically to new followers. Has a `one_time_per_user` flag to prevent repeat sends.
3. **Story Mention Reply** -- Auto-reply when someone mentions your account in their Instagram Story.
4. **Comment Auto-Reply** -- Automatically reply to comments on your posts.

Each slot stores its flow definition as JSON in `_igcr_flow_definition` post meta. The number of automation slots a site can create is limited by the plan's `max_flows` setting.

---

**Q: How do I set up a default DM reply?**

A: Navigate to the automation section for your account. Find the **Default Reply** slot. Click to configure it:
1. Set the **reply type** to "text" (or another supported type like quick reply or template).
2. Enter your **message** text. This is the reply sent when someone DMs you and no keyword rule matches.
3. Save the slot.

The default reply acts as a catch-all fallback. If you have keyword rules configured, they are checked first -- the default reply only fires when none of them match.

---

**Q: How do I create a comment auto-reply?**

A: Navigate to the automation section for your account. Find the **Comment Auto-Reply** slot. Click to configure it:
1. Set the **reply type** (text is most common for comment replies).
2. Enter the **reply message** that will be posted as a reply to incoming comments.
3. Optionally configure keyword filters to only reply to comments containing specific words.
4. Save the slot.

Comment auto-replies are triggered by the `igcr_webhook_comment` action when Instagram sends a comment webhook event.

---

**Q: Can I chain multiple actions together?**

A: Yes. Flow definitions support multiple sequential action steps. Each step has a `type` (e.g., `send_dm`, `reply_comment`, `add_tag`, `wait`) and executes in order. For example, you can create a flow that:
1. Sends a DM reply with a welcome message
2. Waits 5 minutes (`wait` action)
3. Sends a follow-up DM with a product carousel
4. Adds a tag to the user (`add_tag` action)

The flow engine executes steps synchronously. When a `wait` step is encountered, execution pauses and a WP-Cron single event (`igcr_run_flow`) is scheduled to resume after the delay.

---

**Q: How do wait/delay actions work?**

A: The `wait` action type pauses a flow for a specified duration. When the flow engine encounters a `wait` step:
1. It records the current execution state (which step to resume from, the full context).
2. It schedules a single WP-Cron event (`igcr_run_flow`) for the future timestamp (current time + delay).
3. Execution stops.
4. When WP-Cron fires the event, the flow engine resumes from the next step after the wait.

This relies on WP-Cron, which is triggered by site visitors. If your site has low traffic, cron events may fire late. Consider using a system-level cron job (`wget` or `curl` hitting `wp-cron.php` every minute) for more reliable timing.

---

### DM Management

**Q: How do I view DMs from Instagram?**

A: DMs are stored as `igcr_dm` custom post type entries. You can view them through the Gutenberg block-based DM interface on the frontend profile page, or through the WordPress admin under **Posts > DMs** (if the CPT is registered with `show_ui = true`). Each DM thread is a separate post, with individual messages stored as post content or meta. The frontend DM inbox provides a conversation-style interface.

---

**Q: How do I reply to a DM?**

A: From the DM inbox interface, open the conversation thread and type your reply in the message input field. Click **Send**. The plugin calls the Instagram Graph API messaging endpoint via `KDC_QTAP_IGCR_API_Messaging` to deliver your reply. The reply is also stored locally in the DM thread. You can send text messages, quick replies, attachments, generic templates, button templates, and product carousels as reply types.

---

**Q: How do I sync old DM conversations?**

A: DM conversations are populated via webhooks -- the plugin receives new messages in real-time when Instagram sends webhook events. Historical DMs from before the plugin was connected are not automatically imported, as the Instagram Graph API does not provide a bulk DM history endpoint. New DMs will appear as soon as the account is connected and webhook subscription is active.

---

**Q: Are DM senders automatically created as WordPress users?**

A: Yes. When a DM is received from a new Instagram user, the `KDC_QTAP_IGCR_User_Manager` automatically creates a WordPress user account for the sender. The WP user is created with:
- **user_login**: Derived from the Instagram username (sanitized, lowercase)
- **role**: Subscriber
- **user meta**: `_igcr_ig_user_id` and `_igcr_ig_username` are stored for linking

If a WP user already exists for that Instagram user ID (checked via `_igcr_ig_user_id` meta), the existing user is reused. This enables tagging, segmentation, and CRM-like features within WordPress.

---

### Publishing

**Q: How do I publish a WordPress post to Instagram?**

A: When editing a post in the WordPress editor (Gutenberg), look for the **Instagram Account** panel in the sidebar. Select the Instagram account you want to publish to from the dropdown. When you publish (or update) the post:
1. The plugin checks if `_igcr_is_synced` is NOT set (meaning it is an original post, not one imported from Instagram).
2. If `_igcr_account_id` is set, it calls the Instagram Content Publishing API to create the media container and publish it.
3. The post's featured image is used as the Instagram media.

**Important**: If you select an Instagram account but do not attach a featured image (for image posts), the publish will fail. The post must have media content for Instagram.

---

**Q: What media formats are supported for publishing?**

A: Instagram's Content Publishing API supports:
- **Images**: JPEG format recommended. Max file size 8MB. Aspect ratios between 4:5 and 1.91:1.
- **Videos/Reels**: MP4 format, H.264 codec. Max file size 100MB for feed videos, 1GB for Reels. Duration: 3-60 seconds for feed videos, 3-90 seconds for Reels.
- **Carousels**: 2-10 items (images and/or videos) in a single post.

The plugin uses your post's featured image for single image posts.

---

**Q: Can I publish a carousel post?**

A: Carousel publishing depends on the implementation. The plugin uses the Instagram Content Publishing API's carousel endpoint, which requires creating individual media containers for each item, then combining them into a carousel container. Check the post editor for carousel-specific options in the Instagram sidebar panel.

---

**Q: How do I schedule a post for Instagram?**

A: Use WordPress's built-in post scheduling. Set the post's publish date to a future date/time in the editor. When WordPress publishes the post at the scheduled time, the Instagram publish hook fires automatically and publishes to Instagram. The `_igcr_account_id` must be set before scheduling.

---

### Settings

**Q: How do I override the network accent color?**

A: Navigate to **qTap IGcr > Settings** in your site admin. If the site-level settings include an accent color option, you can set a custom color for your subsite that overrides the network default. The network accent color (set at **Network Admin > qTap IGcr > App Settings > Appearance**) applies to all sites unless overridden at the site level.

---

## Editor

### Content

**Q: How do I publish a post to Instagram from the editor?**

A: In the Gutenberg block editor:
1. Open the **post sidebar** (click the Settings icon if not visible).
2. Look for the **Instagram Account** panel below the standard post settings.
3. Select an Instagram account from the dropdown. Only accounts connected to your site appear.
4. Set a featured image for the post (required for Instagram publishing).
5. Write your post content -- the caption text will come from the post content or an excerpt.
6. Click **Publish**.

The Gutenberg sidebar panel is powered by `assets/js/post-ig-sidebar.js` (no build step, uses `wp.element.createElement`).

---

**Q: How do I select which Instagram account to publish to?**

A: In the Gutenberg editor sidebar, the **Instagram Account** panel shows a dropdown of all active Instagram accounts connected to your site. Select the account you want to publish to. The selected account ID is stored in `_igcr_account_id` post meta. If no account is selected and you try to publish, the post will be reverted to draft status (publish gate).

---

**Q: How do I view Instagram comments on a post?**

A: Instagram comments on synced posts are stored as native WordPress comments with `comment_type = 'igcr'`. You can view them:
1. In the WordPress admin under **Comments** -- filter by the post.
2. On the frontend single post page, where comments are rendered like standard WordPress comments.
3. Each comment has meta fields: `_igcr_comment_id` (Instagram comment ID), and parent threading is preserved via `_igcr_ig_parent_id`.

---

**Q: What is the Instagram profile block?**

A: The **Instagram Profile** block (`igcr/instagram-profile`) is a Gutenberg block that displays an Instagram-style profile page for a connected account. It renders the profile picture, username, bio, follower/following counts, and a media grid. Use it by adding the block to any page or post in the editor and selecting the Instagram account to display.

---

**Q: How do I use the post grid block?**

A: The **Instagram Post** block (`igcr/instagram-post`) displays synced Instagram posts in a grid layout. Add it to any page in the Gutenberg editor. Configure the block settings to choose the account, number of posts to display, and layout options. The block queries posts with `_igcr_is_synced = 1` and the selected account's `_igcr_account_id`.

---

## Shop Manager (WooCommerce)

### Products

**Q: How do I connect a product to Instagram?**

A: When editing a WooCommerce product, scroll to the **Product data** section. Under the **General** tab, you will find Instagram-specific fields added by the plugin. Enter the Instagram product tag or catalog information. Click **Update** to save. The plugin stores these fields as product meta. Product sync counts are enforced by the plan's `max_products` limit.

---

**Q: What WooCommerce features are available?**

A: The plugin provides the following WooCommerce integrations:
1. **Product sync fields** -- Meta fields on WooCommerce products for linking to Instagram product tags.
2. **Product carousel automation** -- The `send_product_carousel` flow action sends WooCommerce products as an Instagram DM carousel card.
3. **My Account dashboard** -- The `instagram-accounts` block is rendered on the WooCommerce My Account dashboard page, showing connected Instagram accounts.
4. **Plan-based product limits** -- The `max_products` plan limit controls how many products can be synced.

---

**Q: Are custom order tables (HPOS) supported?**

A: Yes. The plugin declares full compatibility with WooCommerce's High-Performance Order Storage (HPOS / Custom Order Tables) feature. It also declares compatibility with the `product_cache` feature. This is registered via `\Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility()` during `before_woocommerce_init`. You can safely enable HPOS in **WooCommerce > Settings > Advanced > Features** without any conflicts.

---

### Login

**Q: How does Instagram login work on the WooCommerce login form?**

A: The plugin adds a "Continue with Instagram" button to the WordPress login form (`wp-login.php`) via the `login_form` hook. When a user clicks it:
1. They are redirected to Instagram's OAuth authorization page.
2. After approving, Instagram redirects back to the callback URL.
3. The plugin creates a new WordPress user (or finds an existing one linked to that Instagram account).
4. The user is logged in automatically and redirected to the page they came from (or the `redirect_to` parameter).

The login flow differs from the admin account connection flow -- it creates subscriber-level WordPress users for end-user authentication, not for managing Instagram publishing. The button also appears on the standard `wp-login.php` page. You can embed it anywhere using the `[igcr_login_button]` shortcode with optional attributes:
- `label` -- Button text (default: "Continue with Instagram")
- `class` -- CSS class (default: `igcr-login-btn`)
- `return` -- URL to redirect to after login

---

## Troubleshooting (All Personas)

**Q: Token refresh is failing -- what do I do?**

A: Instagram long-lived tokens expire after 60 days. The plugin runs a daily cron (`igcr_token_refresh`) that refreshes all tokens expiring within the next 10 days. If refresh is failing:

1. **Check cron is running**: Go to **Tools > Site Health** and verify WP-Cron is working. Or install a cron monitoring plugin. For production sites, set up a system cron:
   ```
   */5 * * * * wget -q -O /dev/null "https://your-site.com/wp-cron.php"
   ```

2. **Check the activity log**: Go to **qTap IGcr > Activity Log** (site level) or **Network Admin > qTap IGcr > Activity Log** and look for token refresh errors.

3. **Verify AUTH_KEY is stable**: The encryption key is derived from `AUTH_KEY` in `wp-config.php`. If `AUTH_KEY` was changed after tokens were saved, the plugin cannot decrypt them. You will need to disconnect and reconnect all accounts.

4. **Reconnect the account**: If a token has already expired (past the 60-day window), it cannot be refreshed. Disconnect the account and reconnect it via OAuth to get a new token.

---

**Q: Webhooks are not being received -- how do I fix this?**

A: If Instagram events (comments, DMs, mentions) are not arriving:

1. **Verify webhook subscription in Meta Dashboard**: Go to your Meta App > Webhooks > Instagram. Confirm the subscription is active and the callback URL matches `https://your-primary-site.com/wp-json/kdc/v1/instagram/webhook`.

2. **Test the verification endpoint**: Open `https://your-primary-site.com/wp-json/kdc/v1/instagram/webhook?hub_mode=subscribe&hub_verify_token=YOUR_TOKEN&hub_challenge=test` in a browser. You should see a plain response of the challenge value. If you get an error, the verify token does not match.

3. **Check per-account subscription**: After connecting an account, the plugin calls `POST /{ig-user-id}/subscribed_apps` to subscribe to webhook fields. If this call failed during OAuth, Instagram will not send events for that account. Disconnect and reconnect the account to retry.

4. **Check the activity log**: Look for inbound webhook events in the activity log. If events are arriving but not being dispatched, there may be a payload signature mismatch (if you have a webhook secret configured).

5. **Verify SSL**: Instagram requires HTTPS for webhook URLs. Self-signed certificates are not accepted.

6. **Check REST API accessibility**: Ensure `/wp-json/` is accessible and not blocked by a security plugin, `.htaccess` rules, or server firewall.

---

**Q: Media sync is stuck -- how do I unstick it?**

A: If media sync appears to be stuck (no new posts imported, sync button does nothing):

1. **Check the concurrency guard**: The transient `igcr_sync_running_{id}` prevents overlapping runs. It has a 5-minute TTL and should auto-expire. If it is stuck, go to **Network Admin > qTap IGcr > Network Tools** and click **Flush Sync Cursors**. This clears all sync state.

2. **Check the sync cursor**: If `igcr_sync_cursor_{id}` points to an invalid or expired pagination URL, the API returns an error and sync stalls. Flushing cursors resets pagination to page one.

3. **Check API rate limits**: Instagram allows 200 API calls per user per hour. If you have many accounts or are hitting the API frequently, sync calls may fail silently. The plugin enforces a minimum 60-second interval between page fetches (`MIN_PAGE_INTERVAL`).

4. **Check the token**: If the account's token is expired or invalid, all API calls fail. Check **qTap IGcr > Accounts** -- if the token expiry date is in the past, reconnect the account.

---

**Q: OAuth redirect_uri mismatch -- how do I resolve this?**

A: The `redirect_uri` sent to Instagram during OAuth must exactly match one of the URIs configured in your Meta App. To fix:

1. Copy the exact callback URL from your site: `https://your-primary-site.com/wp-json/kdc/v1/instagram/oauth/callback`
2. Go to your Meta App > Instagram Business Login > Settings.
3. Add this exact URL to the **Valid OAuth Redirect URIs** list.
4. Make sure there are no differences in protocol (`http` vs `https`), subdomain (`www` vs non-www), or trailing slashes.
5. Save and try connecting again.

The plugin generates the redirect URI using `KDC_QTAP_IGCR_Network_Settings::primary_rest_url()`, which always uses the primary (main) site's URL. If your primary site's URL changed (e.g., SSL migration), update the Meta App accordingly.

---

**Q: "Plan limit exceeded" error -- what does it mean?**

A: This error means your site has reached the maximum allowed count for a resource defined by your plan:

- **"Account limit reached"**: Your plan's `max_accounts` limit has been reached. You cannot connect more Instagram accounts. Contact your Network Administrator to upgrade your plan or increase the limit.
- **"Flow limit reached"**: Your plan's `max_flows` limit has been reached. Delete unused automation flows or request a plan upgrade.
- **"Post limit reached"**: Your plan's `max_posts` monthly limit has been reached. Wait for the next billing cycle or request a plan upgrade.
- **"Product limit reached"**: Your plan's `max_products` limit has been reached for WooCommerce product sync.

Your Network Administrator can view your plan at **Network Admin > qTap IGcr > Site Assignments** and upgrade it at **Network Admin > qTap IGcr > Plans**.

---

**Q: The plugin shows a PHP version error -- what's required?**

A: qTap IGcr requires **PHP 8.0 or higher**. The plugin uses PHP 8.0+ features including:
- `match` expressions
- Named arguments
- Union types (e.g., `int|string|null`)
- Nullsafe operator (`?->`)

If your PHP version is below 8.0, the plugin shows a compatibility notice via `KDC_QTAP_IGCR_Activator::compat_notice()` and does not load any of its classes. Contact your hosting provider to upgrade PHP to 8.0 or later. The minimum WordPress version required is 6.0.

---

**Q: How do I check if my site's AUTH_KEY is configured correctly?**

A: The plugin uses `AUTH_KEY` from `wp-config.php` to derive the AES-256-CBC encryption key for storing Instagram access tokens. To verify:

1. Open `wp-config.php` on your server.
2. Look for the line `define('AUTH_KEY', '...');`
3. Ensure it contains a unique, random string (not the default placeholder `put your unique phrase here`).
4. **Do not change AUTH_KEY after accounts are connected** -- changing it invalidates all stored encrypted tokens, and you will need to disconnect and reconnect every Instagram account.

If you suspect AUTH_KEY was changed, check whether account connections show errors or token refreshes fail. The solution is to disconnect and reconnect all accounts to re-encrypt tokens with the new key.

---

**Q: Comments from Instagram are not appearing -- why?**

A: Instagram comments are delivered via webhooks and stored as native WordPress comments with `comment_type = 'igcr'`. If they are not appearing:

1. **Check webhook delivery**: Verify webhooks are being received (see the webhooks troubleshooting question above). Comment events arrive via the `comments` webhook field.

2. **Check per-account subscription**: The plugin must call `POST /{ig-user-id}/subscribed_apps` with the `comments` field included. This happens during OAuth connection. Reconnect the account if subscription may have failed.

3. **Check comment moderation**: Instagram comments stored in WordPress go through standard comment moderation rules. Check **Comments > Pending** in your admin for held comments.

4. **Check the activity log**: Look for `igcr_webhook_comment` events. If the webhook fires but comments are not appearing, there may be a deduplication issue (the comment's `_igcr_comment_id` already exists) or a media matching issue (the comment references a post that was not synced).

5. **Legacy comments table**: If you upgraded from an older version, comments may be in the legacy `{prefix}igcr_comments` table. Run the **Migrate Legacy Comments** tool from **Network Admin > qTap IGcr > Network Tools** to move them to native WordPress comments.

---

**Q: The "Continue with Instagram" button is not showing -- why?**

A: The Instagram login button (`[igcr_login_button]` shortcode or `wp-login.php` integration) will not render if:

1. **User is already logged in**: The button only shows for logged-out users. `is_user_logged_in()` returning true suppresses the button.

2. **Meta App not configured**: If the App ID and App Secret are not saved in **Network Admin > qTap IGcr > App Settings**, `KDC_QTAP_IGCR_Network_Settings::is_configured()` returns false and the button is hidden.

3. **Shortcode not processed**: If using the shortcode `[igcr_login_button]`, ensure the page content is processed through WordPress's shortcode parser. Custom page builders may need a shortcode block.

4. **Login form hook not firing**: The `wp-login.php` button is hooked to `login_form`. If a custom login plugin replaces the default login form, this hook may not fire. In that case, use the `[igcr_login_button]` shortcode on your custom login page instead.

To test, log out and visit `wp-login.php` directly. If the button appears there but not on your custom page, the issue is with hook compatibility.
