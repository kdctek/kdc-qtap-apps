# Prompt for `kdc-igcr-wc` — Connector Client Implementation

> Paste the entire section below into a fresh Claude Code session opened **inside the `kdc-igcr-wc` plugin repo**. The prompt is self-contained — it tells the agent the proxy contract, where its endpoints live, the HMAC scheme, and what to build on the client side. The agent should plan first (read the existing plugin), then implement.

---

## Task

You are working on `kdc-igcr-wc`, a WordPress plugin that runs on third-party sites and lets the site owner connect an Instagram Business account through a centralized **Meta OAuth Connector Proxy** hosted at a separate primary qTap WP site. The proxy already exists. **Your job is to implement the client side**: a "Connect Instagram" UI flow plus four small REST endpoints that receive tokens, refreshes, webhook events, and data-deletion notifications from the proxy.

The site owner is **non-technical**. They install the plugin, paste/select the proxy URL once (or it's hard-coded in plugin settings), and click "Connect Instagram". Everything else — initial connection, token storage, 60-day refresh, webhook events — must Just Work without further intervention.

## How the proxy works (you are the client; this is the contract)

**Proxy base URL** is configured in plugin settings as `igcr_wc_proxy_url` (e.g. `https://qtap.example.com`). All proxy paths live under `/wp-json/kdc/v1/connector/...`.

### Flow 1 — Initial connect (TOFU, no credentials yet)

1. User clicks **Connect Instagram** in your plugin admin.
2. Your plugin redirects the browser to:
   ```
   GET {proxy}/wp-json/kdc/v1/connector/oauth/start?site_url={url-encoded https origin of THIS site}&return_url={url-encoded url to land on after success}
   ```
   On a brand-new site you do **not** send `site_key` or `sig` — the proxy will TOFU-register your site and issue credentials in step 6.
3. Proxy redirects user to Meta OAuth consent screen.
4. User authorizes.
5. Meta calls back into the proxy. Proxy exchanges code → 60-day token, stores it, mints a **handover code**, then 302s the browser back to your site at:
   ```
   {your_site}/wp-json/kdc-igcr-wc/v1/connector/receive?igcr_handover={code}&igcr_site_key={key}&return_url={original_return_url}
   ```
   The proxy uses `/wp-json/kdc-igcr-wc/v1/connector/receive` by default. **You must register this exact route.** (If you change it, you also need to send `receive_path` in the start URL — out of scope here, just stick to the default.)
6. Your `receive` endpoint runs **server-side** in the same request (it's a GET hit by the user's browser, but it makes server-to-server POSTs inside its handler):

   **6a. (TOFU only — skip if you already have a stored `site_secret`)** Fetch the site secret:
   ```
   POST {proxy}/wp-json/kdc/v1/connector/oauth/bootstrap
   Content-Type: application/json

   { "site_key": "<igcr_site_key>", "handover_code": "<igcr_handover>" }
   ```
   This is unsigned. Returns `{ "site_secret": "..." }` exactly once per TOFU registration. A 410 means the site is already bootstrapped (you should have the secret stored locally already). Store the secret encrypted.

   **6b. Exchange the handover code for the token:**
   ```
   POST {proxy}/wp-json/kdc/v1/connector/oauth/exchange
   Content-Type: application/json
   X-IGCR-Signature: <hmac>

   {
     "site_key": "<igcr_site_key>",
     "handover_code": "<igcr_handover>",
     "ts": <unix seconds>,
     "nonce": "<random hex>"
   }
   ```
   **Signature canonical:** `site_key + "\n" + handover_code + "\n" + ts + "\n" + nonce`, signed with `hmac_sha256(site_secret, canonical)`. The handover code is single-use and consumed by this call.

7. After exchange, your plugin has:
   - `site_key` (public, store in `wp_options` as `igcr_wc_site_key`)
   - `site_secret` (sensitive, store **encrypted** as `igcr_wc_site_secret_enc` using a class similar to the proxy's `KDC_QTAP_IGCR_Crypto`)
   - `access_token` (the proxy returns it **wrapped** — see "Token unwrap" below — store as `igcr_wc_account_token_enc` keyed by `ig_user_id`)
   - `token_expires_at`, `ig_user_id`, `ig_username`, `name`, `profile_picture_url`, `account_type`, `granted_scopes`

8. Show success UI. From here on, the plugin can call the Instagram Graph API on behalf of the user using the unwrapped token.

### Flow 2 — Subsequent connects (already registered)

Same as Flow 1 but step 2 includes a signed start request:
```
GET {proxy}/wp-json/kdc/v1/connector/oauth/start
  ?site_url={url}
  &return_url={url}
  &site_key={your stored key}
  &ts={unix seconds}
  &nonce={random hex}
  &sig={hmac_sha256(site_secret, "{site_url}\n{site_key}\n{ts}\n{nonce}")}
```
You can reuse your existing `site_key`/`site_secret` for subsequent IG account connections (e.g. connecting a second account).

### Flow 3 — Token refresh push (proxy → you)

The proxy refreshes 60-day tokens on a daily cron and pushes the new token to:
```
POST {your_site}/wp-json/kdc-igcr-wc/v1/connector/receive/refresh
Content-Type: application/json
X-IGCR-Signature: <hmac>

{
  "site_key": "...",
  "ig_user_id": "...",
  "access_token": "<wrapped>",
  "token_expires_at": "YYYY-MM-DD HH:MM:SS",
  "ts": 1234567890,
  "nonce": "..."
}
```

**Signature canonical:** `site_key + "\n" + ig_user_id + "\n" + ts + "\n" + nonce`. Compute `hmac_sha256(site_secret, canonical)` and `hash_equals` against the header. Reject if `abs(time() - ts) > 300`.

On success: unwrap the token (see below), update your stored token + expiry. Return `200 {"ok": true}`. Anything non-2xx will count toward the proxy's failure threshold (5 strikes → site marked inactive there).

### Flow 4 — Webhook event push (proxy → you)

Meta sends webhooks for IG accounts to the proxy's static webhook URL. The proxy detects which entries belong to your site and forwards them to:
```
POST {your_site}/wp-json/kdc-igcr-wc/v1/connector/receive/webhook
Content-Type: application/json
X-IGCR-Signature: <hmac>

{
  "site_key": "...",
  "ig_user_id": "...",
  "object": "instagram",
  "entry": { ... full Meta entry object: id, time, messaging[] or changes[] ... },
  "ts": 1234567890,
  "nonce": "..."
}
```

**Signature canonical:** `site_key + "\n" + ig_user_id + "\n" + ts + "\n" + nonce + "\n" + sha256_hex(json_encode(entry))`. The hash of the entry is included so you can detect tampering.

On verify success: dispatch the entry into your plugin's webhook handling logic (do_action hooks per messaging/changes type, similar to qTap's webhook). Always respond `200 {"ok": true}` quickly — Meta has already gotten its 200 from the proxy, so your timing isn't critical for delivery, but speed still matters because the proxy fires non-blocking with a 1s timeout.

### Token unwrap

Tokens leave the proxy encrypted with **your** `site_secret`. Format: `base64(IV[16] + AES-256-CBC ciphertext)`, key = `substr(sha256(site_secret + "|igcr-connector-token-v1", true), 0, 32)`.

PHP unwrap:
```php
$key = substr( hash( 'sha256', $site_secret . '|igcr-connector-token-v1', true ), 0, 32 );
$raw = base64_decode( $wrapped, true );
$iv  = substr( $raw, 0, 16 );
$ct  = substr( $raw, 16 );
$plain = openssl_decrypt( $ct, 'aes-256-cbc', $key, OPENSSL_RAW_DATA, $iv );
```

### Flow 5 — Data deletion push (proxy → you)

When an end user removes the app from their Instagram settings, Meta calls the proxy's data-deletion endpoint. The proxy applies its configured deletion mode locally and forwards the deletion to you at:
```
POST {your_site}/wp-json/kdc-igcr-wc/v1/connector/receive/data-deletion
Content-Type: application/json
X-IGCR-Signature: <hmac>

{
  "site_key": "...",
  "ig_user_id": "...",
  "mode": "delete" | "deactivate" | "ignore",
  "ts": ...,
  "nonce": "..."
}
```

**Signature canonical:** `site_key + "\n" + ig_user_id + "\n" + ts + "\n" + nonce + "\n" + mode`.

On verify success: mirror the action on your local data — for `delete`, hard-delete the account row + token; for `deactivate`, mark the account inactive but keep the row; for `ignore`, log only. Surface a non-dismissible admin notice ("Instagram revoked access for @username — action: deleted") so the site owner is informed. Respond `200 {"ok": true}` within ~3 seconds; the proxy uses a 5s timeout and tracks acknowledgement status.

### Disconnect

When the user clicks Disconnect in your plugin:
```
POST {proxy}/wp-json/kdc/v1/connector/oauth/disconnect
Content-Type: application/json
X-IGCR-Signature: <hmac>

{
  "site_key": "...",
  "ig_user_id": "...",
  "ts": ...,
  "nonce": "..."
}
```
Canonical: `site_key + "\n" + ig_user_id + "\n" + ts + "\n" + nonce`. On 200, delete local token + account record. The proxy will revoke the Meta permission and mark the account inactive on its side.

## What to implement

1. **Settings page** — single field `Proxy URL` (default `https://qtap.example.com`). Store as `igcr_wc_proxy_url`.

2. **Encryption helper** — a `KDC_IGCR_WC_Crypto` class mirroring the proxy's `KDC_QTAP_IGCR_Crypto` (AES-256-CBC, key derived from `AUTH_KEY`). Used to store `site_secret` and `access_token` at rest.

3. **HMAC helper** — small utility for `sign(secret, canonical)` and `verify(secret, canonical, sig, ts)` with 5-minute clock skew window.

4. **Token wrap/unwrap helper** — implements the AES-256-CBC scheme above. Used by the receive endpoint and by code that needs to call Graph API.

5. **Connect button** — admin UI that opens `/wp-json/kdc/v1/connector/oauth/start?...` in the same window with the appropriate query params (TOFU on first use, signed for subsequent).

6. **Four REST routes** under namespace `kdc-igcr-wc/v1`:
   - `GET /connector/receive` — handles the browser redirect from the proxy. Reads `igcr_handover` + `igcr_site_key` from query. If no `site_secret` is stored locally, calls `/oauth/bootstrap` first to fetch it. Then calls `/oauth/exchange` with HMAC. Stores credentials + token. Redirects the user to `return_url` (validated to belong to this site).
   - `POST /connector/receive/refresh` — refreshes token (verify HMAC, unwrap, persist).
   - `POST /connector/receive/webhook` — receives webhook entry (verify HMAC including entry hash, dispatch via `do_action`).
   - `POST /connector/receive/data-deletion` — handles Meta deletion mirrored from the proxy (verify HMAC, apply `mode`, surface admin notice).

7. **Local data model** — wherever you currently store the IG account, ensure these fields are present and populated from the connector response: `ig_user_id`, `ig_business_id`, `ig_username`, `name`, `profile_picture_url`, `account_type`, `access_token` (encrypted), `token_expires_at`, `granted_scopes`. If the existing plugin already has an account model from an earlier OAuth implementation, **adapt that model** rather than creating a parallel one.

8. **Disconnect button** — calls `/oauth/disconnect` and clears local data.

## Constraints

- The site owner is non-technical. **Zero cron, zero secrets to paste, zero Meta App configuration on their side.**
- Don't ship the Meta App ID/Secret. They live only on the proxy.
- Don't implement your own Meta OAuth flow. The proxy is the only place that talks to Meta for OAuth.
- Don't try to refresh the token yourself. The proxy pushes refreshes. If the receive/refresh endpoint isn't being hit, that's a bug to surface — not a reason to add a local cron.
- Use the existing plugin's coding style (read 3-5 files first to learn it). Don't add comments that just describe what code does.
- If the existing plugin already has an OAuth flow that talks directly to Meta, **rip it out** and replace with the connector flow. Surface this in your final report.

## Report back

After planning + implementing, your final message should include:
1. A list of every file you created or modified (paths + one-line summary).
2. Any spots where the existing plugin needed to be reshaped (e.g. removed direct-OAuth code).
3. Any open dependencies on the proxy (the proxy already exposes `/oauth/start`, `/oauth/bootstrap`, `/oauth/exchange`, `/oauth/disconnect` — only flag dependencies if you discover something missing).
4. The exact manual smoke test the user should run end-to-end:
   - Configure proxy URL → click **Connect Instagram** → land on Meta auth → return to site → verify account stored + token unwraps + Graph API call succeeds.
   - Force-expire the token in the local DB (`token_expires_at = NOW() + INTERVAL 1 DAY`) and trigger the proxy's `igcr_connector_refresh` cron remotely → verify `/refresh` was hit and the local token was rotated.
   - Send a test DM/comment to the connected IG account → verify `/webhook` was hit and dispatched into local handlers.
   - From the Meta App dashboard, simulate a data deletion → verify `/data-deletion` was hit, the local row was actioned per `mode`, and the admin notice surfaced.

Start by listing the plugin's existing structure and any prior OAuth/Instagram code you find. Plan before writing.
