# qTap IGcr REST API Reference

**Plugin**: qTap IGcr (`kdc-qtap-igcr`)
**Version**: 2.50.0+
**Base URL**: `/wp-json/kdc/v1/instagram/`
**Namespace**: `kdc/v1`

---

## Authentication

All authenticated endpoints require one of:

- **WordPress Cookie Authentication** -- for browser-based requests from the WP admin. The `X-WP-Nonce` header must be included with a valid REST nonce (`wp_rest`).
- **Application Passwords** -- for external integrations. Send credentials via HTTP Basic Auth (`Authorization: Basic base64(username:app-password)`).

Public endpoints (webhooks, onboarding checks, data deletion callbacks) use their own verification mechanisms (signed requests, verify tokens, nonces) and do not require WordPress authentication.

---

## Error Format

All errors follow the WordPress `WP_Error` REST response format:

```json
{
  "code": "igcr_forbidden",
  "message": "Access denied.",
  "data": {
    "status": 403
  }
}
```

Common error codes:

| HTTP Status | Code | Description |
|-------------|------|-------------|
| `400` | `rest_invalid_param` | Missing or invalid request parameter |
| `401` | `rest_not_logged_in` | Authentication required |
| `403` | `igcr_forbidden` | Insufficient permissions |
| `404` | `igcr_not_found` | Resource not found |
| `503` | `igcr_not_configured` | Meta App credentials not configured |

---

## Rate Limiting

No application-level rate limiting is enforced by this plugin. Instagram Graph API rate limits apply to outbound API calls (200 calls per user per hour). The onboarding site creation endpoint relies on WordPress nonce verification to prevent abuse.

---

## Endpoints

---

### OAuth

#### `GET /oauth/connect`

Initiate the Instagram Business Login OAuth flow. Redirects the user to Instagram's authorization page.

**Permission**: `manage_options` (for subsite connections) or public (for primary site guest onboarding)

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `blog_id` | integer | No | WordPress site ID for multisite. Determines which subsite the account will be connected to. |
| `return_url` | string | No | URL to redirect back to after OAuth completes. Defaults to the site's Accounts admin page. |

**Example Request**

```bash
curl -X GET "https://ig.cr/wp-json/kdc/v1/instagram/oauth/connect?blog_id=5&return_url=https%3A%2F%2Fexample.ig.cr%2Fwp-admin%2Fadmin.php%3Fpage%3Digcr-accounts" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..."
```

**Response**: `302 Found`

Redirects to Instagram authorization URL with `state={user_id}:{nonce}` and scopes: `instagram_business_basic`, `instagram_business_manage_messages`, `instagram_business_manage_comments`, `instagram_business_content_publish`.

**Error Responses**

| Status | Code | Condition |
|--------|------|-----------|
| `403` | `rest_forbidden` | User is not a site admin for the specified `blog_id` |
| `503` | `igcr_not_configured` | Meta App credentials are not configured in Network Settings |

**Notes**
- The `state` parameter uses the format `{user_id}:{nonce}` for logged-in users, or `g_{random}:{nonce}` for guest flows.
- A transient stores the originating `blog_id` so the callback can switch to the correct subsite context.
- Guest users (not logged in) are allowed on the primary site; a WordPress account is auto-created on callback.

---

#### `GET /oauth/callback`

Handle the Instagram OAuth callback. Exchanges the authorization code for a long-lived token, stores the encrypted account, and subscribes to webhook fields.

**Permission**: Public (state is validated against a stored nonce transient)

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `code` | string | Yes | Authorization code from Instagram |
| `state` | string | Yes | State token matching the one sent in `/oauth/connect` |
| `error` | string | No | Error code from Instagram (if authorization was denied) |
| `error_description` | string | No | Human-readable error description |

**Response**: `302 Found`

Redirects to the originating admin page with `?connected=1` on success, or with `?igcr_error_key={transient_key}` on failure.

**Error Responses**

Errors are stored in a network transient and passed via redirect query parameter rather than returned as JSON.

| Condition | Redirect Query |
|-----------|----------------|
| User denied authorization | `igcr_error_key=igcr_err_{uid}` |
| State expired or invalid | `igcr_error_key=igcr_err_{uid}` |
| Token exchange failed | `igcr_error_key=igcr_err_{uid}` |
| Account limit reached | `igcr_error_key=igcr_err_{uid}` |
| IG account already on another site (unique accounts enforced) | `igcr_error_key=igcr_err_{uid}` |

**Notes**
- After successful connection, the endpoint calls `POST /{ig-user-id}/subscribed_apps` with all supported webhook fields.
- Tokens are encrypted with AES-256-CBC before storage.
- For guest flows, a WordPress user is auto-created from the Instagram profile data.

---

### Webhook

#### `GET /webhook`

Webhook verification endpoint for Meta's hub.challenge handshake.

**Permission**: Public

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `hub_mode` | string | Yes | Must be `subscribe` |
| `hub_verify_token` | string | Yes | Must match the verify token configured in Network Settings |
| `hub_challenge` | string | Yes | Challenge string to echo back |

**Example Request**

```bash
curl -X GET "https://ig.cr/wp-json/kdc/v1/instagram/webhook?hub_mode=subscribe&hub_verify_token=my_token&hub_challenge=1234567890"
```

**Example Response**: `200 OK`

```
1234567890
```

**Error Responses**

| Status | Code | Condition |
|--------|------|-----------|
| `400` | `igcr_bad_mode` | `hub_mode` is not `subscribe` |
| `403` | `igcr_bad_token` | Verify token does not match |

---

#### `POST /webhook`

Receive incoming webhook events from Instagram. Dispatches events to the automation flow engine and persists DMs and comments.

**Permission**: Public (validated via `X-Hub-Signature-256` HMAC-SHA256 signature)

**Headers**

| Name | Required | Description |
|------|----------|-------------|
| `X-Hub-Signature-256` | Yes* | HMAC-SHA256 signature of the request body (* only required when webhook secret is configured) |
| `Content-Type` | Yes | `application/json` |

**Request Body**

Standard Meta webhook payload:

```json
{
  "object": "instagram",
  "entry": [
    {
      "id": "17841400000000000",
      "time": 1700000000,
      "messaging": [...],
      "changes": [
        {
          "field": "comments",
          "value": { ... }
        }
      ]
    }
  ]
}
```

**Example Request**

```bash
curl -X POST "https://ig.cr/wp-json/kdc/v1/instagram/webhook" \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: sha256=abc123..." \
  -d '{"object":"instagram","entry":[{"id":"17841400000000000","time":1700000000,"changes":[{"field":"comments","value":{"id":"12345"}}]}]}'
```

**Example Response**: `200 OK`

```json
{
  "ok": true
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `403` | Invalid `X-Hub-Signature-256` signature |

**Supported Webhook Fields**

| Field | WP Action Hook |
|-------|---------------|
| `comments` | `igcr_webhook_comment` |
| `live_comments` | `igcr_webhook_live_comment` |
| `mentions` | `igcr_webhook_mention` |
| `story_insights` | `igcr_webhook_story` |
| `story_reactions` | `igcr_webhook_story_reaction` |
| `follow` | `igcr_webhook_follow` |
| `comment_poll_response` | `igcr_webhook_comment_poll` |
| `story_poll_response` | `igcr_webhook_story_poll` |
| `share_to_story` | `igcr_webhook_share_to_story` |
| `media` | `igcr_webhook_media` |
| `messages` | `igcr_webhook_dm` |
| `message_reactions` | `igcr_webhook_message_reaction` |
| `messaging_seen` | `igcr_webhook_message_seen` |
| `messaging_postbacks` | `igcr_webhook_postback` |
| `messaging_handover` | `igcr_webhook_handover` |
| `messaging_referral` | `igcr_webhook_referral` |
| `messaging_optins` | `igcr_webhook_optin` |

**Notes**
- On multisite, the `entry.id` (IG user ID) is resolved to the correct subsite via `find_blog_id_by_ig_id()`, and `switch_to_blog()` is called before dispatching.
- If a Node.js microservice is configured, the payload is relayed non-blocking (1s timeout, fire-and-forget).

---

### DM Management

#### `GET /dm/threads`

List DM threads for a connected Instagram account.

**Permission**: `manage_options`

**Query Parameters**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `account_id` | string | Yes | -- | Instagram user ID (numeric string, max 20 chars) |
| `page` | integer | No | `1` | Page number |
| `per_page` | integer | No | `20` | Threads per page (max 50) |

**Example Request**

```bash
curl -X GET "https://example.ig.cr/wp-json/kdc/v1/instagram/dm/threads?account_id=17841400000000000&page=1&per_page=20" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..."
```

**Example Response**: `200 OK`

```json
{
  "threads": [
    {
      "id": 142,
      "sender_ig_id": "17841400000000001",
      "sender_name": "johndoe",
      "last_message": "Hey, is this still available?",
      "last_time": 1700000000,
      "last_direction": "inbound",
      "last_source": "manual",
      "message_count": 5
    }
  ],
  "total": 15,
  "page": 1
}
```

---

#### `GET /dm/messages`

Get all messages in a specific DM thread.

**Permission**: `manage_options`

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `thread_id` | integer | Yes | WordPress post ID of the DM thread (`igcr_dm` CPT) |

**Example Request**

```bash
curl -X GET "https://example.ig.cr/wp-json/kdc/v1/instagram/dm/messages?thread_id=142" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..."
```

**Example Response**: `200 OK`

```json
{
  "thread_id": 142,
  "sender_ig_id": "17841400000000001",
  "sender_name": "johndoe",
  "messages": [
    {
      "id": "aWdfZAG...",
      "text": "Hey, is this still available?",
      "type": "text",
      "direction": "inbound",
      "time": 1700000000
    },
    {
      "id": "local-1700001000",
      "text": "Yes! Check the link in bio.",
      "type": "text",
      "direction": "outbound",
      "time": 1700001000,
      "source": "manual"
    }
  ]
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `404` | Thread not found or wrong post type |

---

#### `POST /dm/send`

Send a DM reply via the Instagram API and record it locally.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `thread_id` | integer | Yes | WordPress post ID of the DM thread |
| `message` | string | Yes | Message text (1--1000 characters) |

**Example Request**

```bash
curl -X POST "https://example.ig.cr/wp-json/kdc/v1/instagram/dm/send" \
  -H "Content-Type: application/json" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..." \
  -d '{"thread_id": 142, "message": "Thanks for reaching out!"}'
```

**Example Response**: `200 OK`

```json
{
  "success": true,
  "message": {
    "id": "aWdfZAG...",
    "text": "Thanks for reaching out!",
    "type": "text",
    "direction": "outbound",
    "time": 1700002000,
    "source": "manual"
  }
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `400` | Thread missing account data |
| `404` | Thread not found |
| `502` | Instagram API error |
| `503` | Account not available (token expired or disconnected) |

---

#### `POST /dm/sync`

Fetch historical DM conversations from the Instagram Conversations API and store them locally.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | string | Yes | Instagram user ID (numeric string) |

**Example Request**

```bash
curl -X POST "https://example.ig.cr/wp-json/kdc/v1/instagram/dm/sync" \
  -H "Content-Type: application/json" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..." \
  -d '{"account_id": "17841400000000000"}'
```

**Example Response**: `200 OK`

```json
{
  "success": true,
  "threads_synced": 8,
  "messages_synced": 47,
  "threads_checked": 25
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `404` | Account not found |
| `502` | Instagram Conversations API error |
| `503` | Account not available |

**Notes**
- Fetches up to 25 conversations with up to 50 messages each.
- Messages are deduplicated by Instagram message ID.
- New WP users are auto-created for unknown DM senders.

---

### Comments

#### `POST /comment`

Post a comment or reply on an Instagram media item and store a local copy.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `post_id` | integer | Yes | WordPress post ID of the synced Instagram post |
| `message` | string | Yes | Comment text (1--2200 characters) |
| `parent_comment_id` | string | No | Instagram comment ID to reply to. Omit for top-level comment. |

**Example Request**

```bash
curl -X POST "https://example.ig.cr/wp-json/kdc/v1/instagram/comment" \
  -H "Content-Type: application/json" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..." \
  -d '{"post_id": 50, "message": "Great photo!", "parent_comment_id": ""}'
```

**Example Response**: `200 OK`

```json
{
  "success": true,
  "comment": {
    "id": 12,
    "text": "Great photo!",
    "ig_comment_id": "17858893269000000",
    "ig_username": "mybrand",
    "is_from_account": true,
    "ig_timestamp": "2024-11-15 10:30:00"
  }
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `400` | Invalid post, missing media ID, or account not linked |
| `503` | Account not available |

**Notes**
- The comment is saved locally first for instant display, then sent to Instagram asynchronously.
- If `parent_comment_id` is provided, the comment is posted as a reply using `POST /{parent_comment_id}/replies`.

---

### Publishing

#### `POST /publish`

Publish media to Instagram from the frontend. Supports single image, reel (video), carousel (album), and story.

**Permission**: `publish_posts`

**Request Body** (`application/json`)

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `account_id` | string | Yes | -- | Instagram user ID of the connected account |
| `caption` | string | Conditional | `""` | Post caption (max 2200 chars). Required for image/reel/carousel, optional for story. |
| `media_type` | string | No | `"image"` | One of: `image`, `reel`, `carousel`, `story` |
| `media_ids` | integer[] | Yes | -- | Array of WordPress attachment IDs. Exactly 1 for image/reel/story, 2--10 for carousel. |
| `draft` | boolean | No | `false` | If `true`, saves as WP draft without publishing to Instagram |
| `location_id` | string | No | `""` | Facebook Page ID with location data (not for stories) |
| `collaborators` | string[] | No | `[]` | Up to 3 Instagram usernames for collaborator invites (not for stories) |
| `alt_text` | string | No | `""` | Alt text for images (max 1000 chars, images only) |
| `share_to_feed` | boolean | No | `true` | For reels: also share to Feed |

**Example Request**

```bash
curl -X POST "https://example.ig.cr/wp-json/kdc/v1/instagram/publish" \
  -H "Content-Type: application/json" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..." \
  -d '{
    "account_id": "17841400000000000",
    "caption": "Check out our new collection! #fashion",
    "media_type": "image",
    "media_ids": [301],
    "alt_text": "Model wearing blue dress"
  }'
```

**Example Response (publish)**: `200 OK`

```json
{
  "success": true,
  "post_id": 155,
  "ig_media_id": "17899506342000000",
  "permalink": "https://www.instagram.com/p/ABC123/",
  "wp_url": "https://example.ig.cr/post/check-out-our-new-collection/"
}
```

**Example Response (draft)**: `201 Created`

```json
{
  "success": true,
  "post_id": 156,
  "is_draft": true,
  "wp_url": "https://example.ig.cr/?p=156"
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `400` | Invalid image/video, wrong number of media items for type, caption too long |
| `502` | Instagram API rejected the publish request |
| `503` | Account not available |

---

#### `POST /publish-draft`

Publish an existing WP draft post to Instagram.

**Permission**: `publish_posts`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `post_id` | integer | Yes | WordPress post ID of the draft to publish |

**Example Request**

```bash
curl -X POST "https://example.ig.cr/wp-json/kdc/v1/instagram/publish-draft" \
  -H "Content-Type: application/json" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..." \
  -d '{"post_id": 156}'
```

**Example Response**: `200 OK`

```json
{
  "success": true,
  "post_id": 156,
  "ig_media_id": "17899506342000001",
  "permalink": "https://www.instagram.com/p/DEF456/",
  "wp_url": "https://example.ig.cr/post/check-out-our-new-collection/"
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `400` | Media file not found, carousel requires at least 2 images |
| `404` | Draft post not found |
| `502` | Instagram API error |
| `503` | Account not available |

---

### Automation (Account-Level)

#### `GET /automation`

List all automation slots for an account. Auto-creates missing built-in slots on first access.

**Permission**: `manage_options`

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |

**Example Request**

```bash
curl -X GET "https://example.ig.cr/wp-json/kdc/v1/instagram/automation?account_id=1" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..."
```

**Example Response**: `200 OK`

```json
{
  "slots": [
    {
      "id": 200,
      "slot": "welcome_dm",
      "title": "Welcome DM",
      "description": "Send a message when someone DMs you for the first time",
      "icon": "message-circle",
      "is_active": true,
      "config": {
        "version": 3,
        "reply_type": "text",
        "message": "Hey! Thanks for reaching out."
      },
      "run_count": 42,
      "is_builtin": true
    }
  ],
  "keyword_rules": [
    {
      "id": 210,
      "slot": "keyword_rule",
      "title": "Price Request",
      "is_active": true,
      "config": {
        "version": 3,
        "reply_type": "text",
        "message": "Check the link!",
        "keywords": ["price", "cost", "how much"],
        "keyword_match_mode": "contains"
      },
      "run_count": 15,
      "is_builtin": false
    }
  ]
}
```

---

#### `PUT /automation/{id}`

Update an automation slot's configuration, title, or active state.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | integer | Yes | Flow post ID |

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `title` | string | No | New title for the slot |
| `config` | object | No | Updated configuration object |
| `is_active` | boolean | No | Set active state |

**Example Request**

```bash
curl -X PUT "https://example.ig.cr/wp-json/kdc/v1/instagram/automation/200" \
  -H "Content-Type: application/json" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..." \
  -d '{"config": {"version": 3, "reply_type": "text", "message": "Updated welcome!"}}'
```

**Example Response**: `200 OK`

```json
{
  "success": true
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `404` | Slot not found |

---

#### `PUT /automation/{id}/toggle`

Toggle an automation slot between active and inactive. Cascades to chained keyword rules.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | integer | Yes | Flow post ID |

**Example Response**: `200 OK`

```json
{
  "success": true,
  "is_active": true
}
```

---

#### `POST /automation/keyword`

Create a new keyword rule for an account.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |
| `title` | string | No | Rule title (default: "Keyword Rule") |

**Example Response**: `201 Created`

```json
{
  "success": true,
  "rule": {
    "id": 215,
    "slot": "keyword_rule",
    "title": "Keyword Rule",
    "description": "",
    "icon": "key",
    "is_active": false,
    "config": {
      "version": 3,
      "reply_type": "text",
      "message": "",
      "keywords": [],
      "keyword_match_mode": "contains"
    },
    "run_count": 0,
    "is_builtin": false
  }
}
```

---

#### `DELETE /automation/{id}`

Delete a keyword rule. Built-in slots cannot be deleted.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | integer | Yes | Flow post ID |

**Example Response**: `200 OK`

```json
{
  "success": true
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `403` | Attempted to delete a built-in slot |
| `404` | Slot not found |

---

#### `POST /automation/{id}/chain`

Create a chained keyword rule triggered by a postback button payload. Maximum chain depth: 3.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | integer | Yes | Parent flow post ID |

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `payload` | string | Yes | Button payload string that triggers this chain |
| `account_id` | integer | No | Account ID (defaults to parent's account) |

**Example Response**: `201 Created`

```json
{
  "success": true,
  "chain": {
    "id": 220,
    "slot": "keyword_rule",
    "title": "Price Request \u2192 color_blue",
    "is_active": true,
    "config": {
      "version": 3,
      "reply_type": "text",
      "message": "",
      "keywords": ["color_blue"],
      "keyword_match_mode": "equals"
    },
    "run_count": 0,
    "is_builtin": false,
    "chained_from": 210
  }
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `400` | Missing payload or max chain depth (3) reached |
| `404` | Parent slot not found |

---

#### `DELETE /automation/{id}/chain`

Delete a specific chain by payload string.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | integer | Yes | Parent flow post ID |

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `payload` | string | Yes | Payload of the chain to delete |

**Example Response**: `200 OK`

```json
{
  "success": true
}
```

---

#### `GET /automation/{id}/versions`

List configuration version history for an automation slot.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | integer | Yes | Flow post ID |

**Example Response**: `200 OK`

```json
{
  "success": true,
  "versions": [
    {
      "index": 0,
      "config": { "version": 3, "reply_type": "text", "message": "Hello!" },
      "saved_at": "2024-11-15T10:30:00Z"
    },
    {
      "index": 1,
      "config": { "version": 3, "reply_type": "text", "message": "Hi there!" },
      "saved_at": "2024-11-16T14:00:00Z"
    }
  ]
}
```

---

#### `POST /automation/{id}/restore`

Restore a previous configuration version.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `id` | integer | Yes | Flow post ID |

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `version_index` | integer | Yes | Index of the version to restore |

**Example Response**: `200 OK`

```json
{
  "success": true,
  "config": {
    "version": 3,
    "reply_type": "text",
    "message": "Hello!"
  }
}
```

---

### Automation (Post-Level)

#### `GET /automation/post/{post_id}`

List post-level automation slots and available templates for a specific Instagram post.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `post_id` | integer | Yes | WordPress post ID of the Instagram post |

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |

**Example Response**: `200 OK`

```json
{
  "slots": [
    {
      "id": 300,
      "slot": "post_comment_reply",
      "title": "Comment Auto-Reply",
      "is_active": false,
      "config": { "version": 3, "reply_type": "text", "message": "" }
    }
  ],
  "keyword_rules": [],
  "templates": [
    {
      "key": "giveaway",
      "title": "Giveaway Entry",
      "description": "Auto-reply to comments and DM a confirmation"
    }
  ]
}
```

---

#### `POST /automation/post/{post_id}/keyword`

Create a post-level keyword rule.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `post_id` | integer | Yes | WordPress post ID |

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |

**Example Response**: `201 Created`

```json
{
  "success": true,
  "rule": {
    "id": 310,
    "slot": "post_keyword_rule",
    "title": "Keyword Rule",
    "is_active": false,
    "config": {
      "version": 3,
      "reply_type": "text",
      "message": "",
      "keywords": [],
      "keyword_match_mode": "contains",
      "also_like_comment": false,
      "dm_followup": false
    },
    "run_count": 0,
    "is_builtin": false,
    "post_id": 50
  }
}
```

---

#### `POST /automation/post/{post_id}/template`

Apply a preset automation template to a post.

**Permission**: `manage_options`

**URL Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `post_id` | integer | Yes | WordPress post ID |

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |
| `template_key` | string | Yes | Template identifier (e.g., `giveaway`) |

**Example Response**: `201 Created`

```json
{
  "success": true,
  "rule": {
    "id": 311,
    "slot": "post_keyword_rule",
    "title": "Giveaway Entry",
    "description": "Auto-reply to comments and DM a confirmation",
    "is_active": false,
    "config": { "..." : "..." },
    "run_count": 0,
    "is_builtin": false,
    "post_id": 50
  }
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `404` | Template key not found |

---

#### `GET /automation/posts`

Paginated list of Instagram posts that have post-level automations configured.

**Permission**: `manage_options`

**Query Parameters**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `account_id` | integer | Yes | -- | Internal account row ID |
| `page` | integer | No | `1` | Page number |
| `per_page` | integer | No | `10` | Posts per page (max 50) |

**Example Response**: `200 OK`

```json
{
  "posts": [
    {
      "post_id": 50,
      "title": "Summer collection launch",
      "url": "https://example.ig.cr/post/summer-collection/",
      "thumbnail": "https://example.ig.cr/wp-content/uploads/2024/11/summer-150x150.jpg",
      "total_rules": 3,
      "active_rules": 2
    }
  ],
  "page": 1,
  "per_page": 10,
  "total": 5,
  "total_pages": 1
}
```

---

### Persistent Menu & Ice Breakers

#### `GET /automation/menu`

Get the current Instagram persistent menu for an account.

**Permission**: `manage_options`

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |

**Example Response**: `200 OK`

```json
{
  "items": [
    {
      "type": "postback",
      "title": "Shop Now",
      "payload": "SHOP_NOW"
    },
    {
      "type": "web_url",
      "title": "Visit Website",
      "url": "https://example.com"
    }
  ]
}
```

---

#### `PUT /automation/menu`

Set the persistent menu for an account via Instagram API.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |
| `items` | array | Yes | Array of menu items (postback or web_url) |

**Example Response**: `200 OK`

```json
{
  "success": true
}
```

---

#### `DELETE /automation/menu`

Delete the persistent menu for an account.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |

**Example Response**: `200 OK`

```json
{
  "success": true
}
```

---

#### `GET /automation/ice-breakers`

Get current ice breaker questions for an account.

**Permission**: `manage_options`

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |

**Example Response**: `200 OK`

```json
{
  "questions": [
    {
      "question": "What products do you have?",
      "payload": "PRODUCTS"
    },
    {
      "question": "How can I place an order?",
      "payload": "ORDER_HELP"
    }
  ]
}
```

---

#### `PUT /automation/ice-breakers`

Set ice breaker questions via Instagram API.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |
| `questions` | array | Yes | Array of question objects with `question` and `payload` keys |

**Example Response**: `200 OK`

```json
{
  "success": true
}
```

---

#### `DELETE /automation/ice-breakers`

Delete all ice breaker questions for an account.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `account_id` | integer | Yes | Internal account row ID |

**Example Response**: `200 OK`

```json
{
  "success": true
}
```

---

### Node.js Bridge

These endpoints are used by the Node.js microservice for flow action execution and logging. They require `manage_options` permission (authenticated via Application Password from the Node service).

#### `POST /flow-action`

Execute a single automation action (send DM, reply comment, manage tags).

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `action` | string | Yes | Action type: `send_dm`, `send_quick_reply`, `reply_comment`, `like_comment`, `add_tag`, `remove_tag` |
| `account_id` | integer | Yes | Internal account row ID |
| `recipient_id` | string | Conditional | Instagram PSID (for `send_dm`, `send_quick_reply`) |
| `message` | string | Conditional | Message text (for `send_dm`, `send_quick_reply`, `reply_comment`) |
| `quick_replies` | array | No | Quick reply buttons (for `send_quick_reply`) |
| `media_id` | string | Conditional | Instagram media ID (for `reply_comment`) |
| `comment_id` | string | Conditional | Instagram comment ID (for `reply_comment`, `like_comment`) |
| `wp_user_id` | integer | Conditional | WordPress user ID (for `add_tag`, `remove_tag`) |
| `tag` | string | Conditional | Tag string (for `add_tag`, `remove_tag`) |

**Example Response**: `200 OK`

```json
{
  "ok": true,
  "message_id": "aWdfZAG..."
}
```

---

#### `POST /flow-log`

Log flow execution start, completion, or failure.

**Permission**: `manage_options`

**Request Body** (`application/json`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `action` | string | Yes | `start`, `complete`, or `fail` |
| `flow_id` | integer | Conditional | Flow post ID (for `start`) |
| `account_id` | integer | Conditional | Account ID (for `start`) |
| `trigger` | string | Conditional | Trigger name (for `start`) |
| `context` | object | No | Trigger context data (for `start`) |
| `log_id` | integer | Conditional | Flow log ID (for `complete` and `fail`) |
| `reason` | string | No | Failure reason (for `fail`) |

**Example Response (start)**: `200 OK`

```json
{
  "log_id": 42
}
```

**Example Response (complete/fail)**: `200 OK`

```json
{
  "ok": true
}
```

---

### Onboarding

#### `GET /onboarding/check`

> **Note**: This endpoint is registered under namespace `kdc/v1`, at path `/onboarding/check` (not under `/instagram/`).

Check subdomain availability for new site creation.

**Permission**: Public

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `slug` | string | Yes | Desired subdomain (2--50 chars, alphanumeric and hyphens only) |

**Example Request**

```bash
curl -X GET "https://ig.cr/wp-json/kdc/v1/onboarding/check?slug=johndoe"
```

**Example Response**: `200 OK`

```json
{
  "available": true,
  "suggested": "johndoe"
}
```

**Unavailable Response**:

```json
{
  "available": false,
  "suggested": "admin",
  "reason": "reserved"
}
```

**Reasons**: `too_short` (slug < 2 chars), `reserved` (blocked subdomain list).

**Blocked Subdomains**: `www`, `mail`, `smtp`, `ftp`, `admin`, `api`, `app`, `help`, `support`, `blog`, `static`, `cdn`, `media`, `assets`, `ig`, `igcr`, `instagram`, `meta`, `join`, `signup`, `login`, `dashboard`, `network`, `test`, `demo`, `staging`, `ns1`, `ns2`, `ns3`, `cpanel`, `webmail`, `ssl`.

---

#### `POST /onboarding/create`

> **Note**: This endpoint is registered under namespace `kdc/v1`, at path `/onboarding/create` (not under `/instagram/`).

Provision a new WordPress subsite from a connected Instagram account. The account's encrypted token is moved to the new subsite -- no second OAuth round-trip needed.

**Permission**: Must be logged in (verified via WordPress nonce)

**Request Body** (`application/json` or form-encoded)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `ig_account_id` | integer | Yes | Row ID from primary site's `igcr_accounts` table |
| `igcr_nonce` | string | Yes | WordPress nonce for `igcr_ob_create` |

**Example Response**: `200 OK`

```json
{
  "redirect_url": "https://ig.cr/onboard/?igcr_ob_done=1&igcr_site=5"
}
```

**Error Responses**

| Status | Error Code | Condition |
|--------|------------|-----------|
| `403` | `not_logged_in` | User is not logged in |
| `403` | `bad_nonce` | Invalid or expired nonce |
| `404` | `account_not_found` | Account not found or inactive |
| `422` | `invalid_account` | Missing `ig_account_id` |
| `422` | `invalid_subdomain` | Derived subdomain too short |
| `422` | `subdomain_reserved` | Subdomain is on the blocked list |
| `422` | `subdomain_taken` | Subdomain already in use |
| `500` | `site_create_failed` | `wpmu_create_blog()` failed |

**Notes**
- The subdomain is derived from the Instagram username (periods stripped, lowercased).
- A free plan is auto-assigned to the new site.
- If a template site is configured in Network Settings, its content is cloned.
- Webhook subscriptions are activated for the new subsite.

---

### Data Deletion

#### `POST /data-deletion`

Meta data deletion callback. Called by Instagram when a user removes the app from their account settings.

**Permission**: Public (verified via `signed_request` HMAC-SHA256 using the app secret)

**Request Body** (`application/x-www-form-urlencoded`)

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `signed_request` | string | Yes | Meta signed request (base64url-encoded signature + JSON payload) |

**Example Response**: `200 OK`

```json
{
  "url": "https://ig.cr/wp-json/kdc/v1/instagram/data-deletion/status?code=igcr_del_abc123xyz",
  "confirmation_code": "igcr_del_abc123xyz"
}
```

**Error Responses**

| Status | Code | Condition |
|--------|------|-----------|
| `400` | `missing_signed_request` | No `signed_request` parameter |
| `400` | `invalid_signed_request` | Malformed signed request format |
| `400` | `missing_user_id` | No `user_id` in signed request payload |
| `403` | `invalid_signature` | Signature verification failed |

**Notes**
- The action taken depends on the Network Settings deletion mode: `delete` (permanently removes account data), `deactivate` (disconnects account, preserves data), or `ignore` (logs only).
- An email notification is sent to the site admin and network admin.
- A persistent admin notice appears until acknowledged.
- The confirmation code is stored as a network transient for 30 days.

---

#### `GET /data-deletion/status`

Check the status of a data deletion request. Meta polls this URL after receiving the deletion callback response.

**Permission**: Public

**Query Parameters**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `code` | string | Yes | Confirmation code from the deletion callback |

**Example Request**

```bash
curl -X GET "https://ig.cr/wp-json/kdc/v1/instagram/data-deletion/status?code=igcr_del_abc123xyz"
```

**Example Response**: `200 OK`

```json
{
  "status": "deleted",
  "ig_user_id": "17841400000000000",
  "mode": "delete",
  "completed_at": "2024-11-15T10:30:00+00:00"
}
```

**Error Responses**

| Status | Condition |
|--------|-----------|
| `404` | Confirmation code expired (>30 days) or invalid |

---

### Realtime

#### `GET /realtime/token`

> **Note**: This endpoint is registered under namespace `kdc/v1`, at path `/realtime/token` (not under `/instagram/`).

Generate a short-lived JWT token for authenticating SSE connections to the Node.js microservice.

**Permission**: Must be logged in

**Example Request**

```bash
curl -X GET "https://example.ig.cr/wp-json/kdc/v1/realtime/token" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..."
```

**Example Response**: `200 OK`

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expires_in": 900,
  "node_url": "https://node.ig.cr"
}
```

**Error Responses**

| Status | Code | Condition |
|--------|------|-----------|
| `401` | `rest_not_logged_in` | User is not authenticated |
| `503` | `igcr_node_not_configured` | Node.js microservice URL/secret not set |

**JWT Payload Claims**

| Claim | Description |
|-------|-------------|
| `sub` | WordPress user ID |
| `blog_id` | Current WordPress site ID |
| `iat` | Issued-at timestamp |
| `exp` | Expiry timestamp (iat + 900s) |
| `iss` | `igcr-wp` |

**Notes**
- Token is signed with HMAC-SHA256 using the `node_secret` shared between WordPress and the Node.js microservice.
- Clients should refresh the token before the 15-minute expiry.
- Use the token with the SSE endpoint: `GET {node_url}/events/{blogId}/{accountId}?token={token}`

---

### Events (Conditional)

#### `GET /events`

> **Note**: This endpoint is registered under namespace `kdc/v1/instagram`, at path `/events`.

List events from the kdc-qtap-events plugin or The Events Calendar / Event Tickets Plus. Used by the automation event picker.

**Permission**: `manage_options`

**Example Request**

```bash
curl -X GET "https://example.ig.cr/wp-json/kdc/v1/instagram/events" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..."
```

**Example Response**: `200 OK`

```json
{
  "events": [
    {
      "id": 45,
      "title": "Summer Music Festival",
      "date": "2024-12-15",
      "venue": "Mumbai",
      "price": 500,
      "image_url": "https://example.ig.cr/wp-content/uploads/2024/11/festival-300x200.jpg",
      "url": "https://example.ig.cr/event/summer-music-festival/",
      "reg_url": "https://example.ig.cr/event/summer-music-festival/"
    }
  ]
}
```

**Notes**
- Returns an empty array if neither kdc-qtap-events nor The Events Calendar is active.
- Fetches up to 50 active/published events.

---

### WooCommerce Products

#### `GET /products`

List WooCommerce products for the automation product carousel picker.

**Permission**: `manage_options`

**Example Request**

```bash
curl -X GET "https://example.ig.cr/wp-json/kdc/v1/instagram/products" \
  -H "X-WP-Nonce: abc123" \
  --cookie "wordpress_logged_in_xxx=..."
```

**Example Response**: `200 OK`

```json
{
  "products": [
    {
      "id": 100,
      "title": "Blue T-Shirt",
      "price": "$29.99",
      "image_url": "https://example.ig.cr/wp-content/uploads/2024/11/shirt.jpg",
      "permalink": "https://example.ig.cr/product/blue-t-shirt/",
      "slug": "blue-t-shirt",
      "type": "simple"
    },
    {
      "id": 101,
      "title": "Classic Hoodie",
      "price": "$49.99 - $59.99",
      "image_url": "https://example.ig.cr/wp-content/uploads/2024/11/hoodie.jpg",
      "permalink": "https://example.ig.cr/product/classic-hoodie/",
      "slug": "classic-hoodie",
      "type": "variable",
      "variations": [
        {
          "id": 102,
          "title": "Small / Black",
          "price": "$49.99",
          "image_url": "https://example.ig.cr/wp-content/uploads/2024/11/hoodie-black.jpg"
        }
      ]
    }
  ]
}
```

**Notes**
- Returns an empty array if WooCommerce is not active.
- Fetches up to 50 published products, sorted by title ascending.
- Variable products include their available variations.

---

### Post Grid (Block Data)

#### `GET /posts`

Paginated Instagram post grid data for the `qtap/instagram-post` Gutenberg block. Returns rendered HTML cards and pagination metadata.

**Permission**: Public

**Query Parameters**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `page` | integer | No | `1` | Page number (minimum 1) |
| `per_page` | integer | No | `12` | Posts per page (max 48) |
| `total_posts` | integer | No | `-1` | Cap total available posts (-1 = unlimited) |
| `columns` | integer | No | `6` | Grid columns (`3` or `6`) |
| `types` | string | No | `""` | Comma-separated content type filter: `ig-image`, `ig-video`, `ig-album`, `ig-reel` |
| `show_caption` | boolean | No | `true` | Include caption text in cards |
| `show_date` | boolean | No | `false` | Include date in cards |
| `show_badge` | boolean | No | `true` | Include media type badge (Video/Album) |

**Example Request**

```bash
curl -X GET "https://example.ig.cr/wp-json/kdc/v1/instagram/posts?page=1&per_page=12&types=ig-image,ig-reel"
```

**Example Response**: `200 OK`

```json
{
  "html": "<article class=\"igcr-post-card igcr-post-card--image\">...</article>...",
  "page": 1,
  "total_pages": 5,
  "found_posts": 58
}
```

**Notes**
- Returns server-rendered HTML cards (not raw JSON data) for progressive enhancement.
- Posts are ordered by `_igcr_timestamp` descending (newest first).
- Only posts with `_igcr_is_synced = 1` are included.

---

## Changelog

| Version | Changes |
|---------|---------|
| 2.50.0 | Added data deletion endpoint, tabbed network settings |
| 2.46.0 | Added publish-draft endpoint, post-level automation |
| 2.44.0 | Added persistent menu, ice breakers, Node.js bridge endpoints |
| 2.43.0 | Added DM management, comment, publish, flow, and realtime endpoints |
| 2.0.0 | Initial REST API: OAuth, webhook, onboarding |
