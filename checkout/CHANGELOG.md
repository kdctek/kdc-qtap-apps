# Changelog

All notable changes to qTap Checkout are documented in this file.

## [1.0.2] - 2026-04-28

### Removed — `KDC_qTap_Checkout_Cart_FAB` (introduced in v1.0.1, lifetime ~minutes)

Per the consolidation rule announced in the v3.0.4/3.0.5 parent release cycle: **all FAB renderers belong in the parent plugin (`kdc-qtap`), not split across the suite.** v1.0.1 had moved the WooCommerce Cart FAB from `kdc-qtap-mobile` into this plugin "because cart-related code lives here" — but that broke the single-home rule the user pinned down immediately afterwards. The Cart FAB is now in parent v3.0.5+ as [`KDC_qTap_Cart_FAB`](../kdc-qtap/includes/class-kdc-qtap-cart-fab.php).

**Deleted:** `includes/class-kdc-qtap-checkout-cart-fab.php`, plus its `require_once` and `init_components()` line in the bootstrap.

**Deploy:** parent v3.0.5 first (so the Cart FAB exists in its new home), then this plugin.

**Filter compatibility:** `kdc_qtap_checkout_cart_fab_color` is still honoured by the parent's `KDC_qTap_Cart_FAB::get_accent_color()` — sites that hooked it during the v1.0.1 window keep working. The canonical filter going forward is `kdc_qtap_cart_fab_color`.

## [1.0.1] - 2026-04-28

### New — `KDC_qTap_Checkout_Cart_FAB`

The WooCommerce **Cart FAB** (floating shortcut button to checkout, visible when the user has items in their cart) moved here from `kdc-qtap-mobile` where it had been living for incidental reasons (mobile happened to own the WC My Account integration). Cart-related UI belongs with the cart plugin.

**File added:** [`includes/class-kdc-qtap-checkout-cart-fab.php`](includes/class-kdc-qtap-checkout-cart-fab.php)

Behaviour preserved verbatim from kdc-qtap-mobile v2.15.1:
- Registers with the parent FAB registry (`kdc-qtap` v3.0.0+) at `kdc_qtap_loaded:5`
- Default anchor: bottom-right
- Default scope: `wc_my_account`, `dashboard`, `cart_has_items`
- Hides itself when the cart is empty or the user is already on `/cart` or `/checkout`
- Lifts above the mobile bottom-nav (extra `bottom: 80px`) on dashboard / WC My Account pages
- Accent color resolution: filter override → WC checkout-button color → WC email base color → WP-admin blue

**New filter:** `kdc_qtap_checkout_cart_fab_color` lets sites override the accent color.

**Back-compat:** the legacy `kdc_qtap_mobile_cart_fab_color` filter is still honoured (checked second when the new filter returns empty), so existing sites that overrode the color don't lose their customization.

### Coordinated release

This release ships in lockstep with:
- **kdc-qtap v3.0.4** — adds `KDC_qTap_FAB_Menu` (the qTap dashboard menu FAB also moved out of mobile to its proper home in the parent).
- **kdc-qtap-mobile v2.15.2** — strips both FAB renderers, helpers, and the legacy `wp_footer` fallbacks.

**Deploy order:** parent → this plugin → mobile, so the new FABs exist before mobile drops the old ones.

## [1.0.0] - 2026-04-27

### Added — Initial release

Cart, checkout router, gateway base + registry, payment completion, and the Razorpay + Zaakpay gateways. Code originally shipped briefly inside `kdc-qtap` v3.0.0 then `kdc-qtap-finance` v3.19.0; split into a standalone sibling plugin so any qTap child (Finance, Events, future modules) can opt in without taking Finance as a hard dependency.

#### Modules

- `KDC_qTap_Checkout_Cart` — line-item store keyed by `source + source_id` in user_meta. Helper: `kdc_qtap_checkout_cart()`.
- `KDC_qTap_Checkout_Router` — reserves the dashboard's `checkout` panel, handles `/dashboard/checkout/`, `/checkout/pay/{token}/`, `/checkout/return/{token}/`. Constant `KDC_qTap_Checkout_Router::WEBHOOK_PATH = 'qtap-checkout/webhook/'` keeps the server-to-server webhook URL stable across dashboard slug changes.
- `KDC_qTap_Checkout_Payment_Gateway` — abstract gateway base. Concrete gateways implement `process()`, `handle_return()`, `handle_webhook()`, `render_settings_fields()`, `sanitize_settings()`.
- `KDC_qTap_Checkout_Payment_Gateways` — registry. Filter: `kdc_qtap_checkout_payment_gateways`. Helpers: `kdc_qtap_checkout_payment_gateways()`, `kdc_qtap_checkout_register_payment_gateway()`.
- `KDC_qTap_Checkout_Payment_Completion` — token + idempotency for the `kdc_qtap_checkout_paid` action.
- `KDC_qTap_Checkout_Gateway_Razorpay` — Standard Checkout JS modal, HMAC-SHA256 return signature, `X-Razorpay-Signature` webhook.
- `KDC_qTap_Checkout_Gateway_Zaakpay` — hosted-checkout auto-submit form, alpha-sorted-pipe-concat HMAC-SHA256 checksum.
- `KDC_qTap_Checkout_Admin` — settings page at qTap App > Checkout. Payment backend radio (WooCommerce vs qTap Cart + Gateway) + per-gateway accordion.

#### Activation migration

Copies pre-existing finance v3.19.0 keys to the new `kdc_qtap_checkout_*` keys if the new keys don't exist:

- option `kdc_qtap_finance_payment_backend` → `kdc_qtap_checkout_backend`
- option `kdc_qtap_finance_gateway_{id}` → `kdc_qtap_checkout_gateway_{id}`

User-meta cart key (`kdc_qtap_finance_cart`) is **not** migrated — carts have a 12h TTL and were extremely unlikely to be populated during the brief v3.19.x window.

#### Stable webhook URL

S2S webhook lives at `/qtap-checkout/webhook/{gateway_id}/`. This replaces the brief `/qtap-pay/webhook/{id}/` path that shipped in parent v3.0.0. Sites that configured a merchant dashboard during the few hours that path was live need to update the URL in their gateway merchant config — flagged in release notes.

#### Public hook names

- `kdc_qtap_checkout_paid` — fires exactly once per paid cart_token. Args: `( $items, $user_id, $context )`.
- `kdc_qtap_checkout_item_added` — fires when a line item is added.
- `kdc_qtap_checkout_item_removed` — fires when a line item is removed.
- `kdc_qtap_checkout_cleared` — fires when the cart is emptied.

#### Required parent

- `kdc-qtap` (qTap App) v3.0.0+
