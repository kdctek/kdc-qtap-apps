# qTap Page Loader — Child Plugin Integration Guide

**Parent plugin:** qTap App (`kdc-qtap` v2.7.6+)
**For:** AI assistants and developers building child plugins (mobile, finance, events, kyc, etc.) that have frontend blocks doing async transactions.

---

## What is the Page Loader?

A full-screen overlay with a centered spinner card that blocks the entire page during async transactions. Use it when:

- The user is paying for something
- Verifying an OTP
- Saving data and redirecting
- Any operation where premature clicks/navigation would cause inconsistent state

**Visual:** dimmed/blurred backdrop + white card with spinner + message. Locks body scroll. Sits above the WP admin bar (`z-index: 999999`).

---

## When NOT to Use It

| Situation | Use this instead |
|-----------|------------------|
| Loading a list/section | `KdcQtapUI.showLoading($container, msg)` (container-level) |
| Submitting one button click | `KdcQtapUI.setButtonLoading($btn, true, msg)` |
| Showing a result message | `KdcQtapUI.showMessage($container, msg, 'success')` |
| Skeleton placeholder while data loads | Container-level loading |

The page loader is a **blocking** UX. Reserve it for transactions the user must wait through, not for routine fetches.

---

## Quick Start

### 1. Make sure parent helpers are enqueued

In your block's `render_callback()` or wherever you enqueue your frontend block JS:

```php
public function render_block( $attributes, $content, $block ) {
    // Load shared CSS + JS (idempotent — safe to call multiple times).
    if ( function_exists( 'kdc_qtap_enqueue_frontend_components' ) ) {
        kdc_qtap_enqueue_frontend_components();
    }
    // ... your block render ...
}
```

When enqueueing your block's frontend JS, declare `kdc-qtap-frontend-helpers` as a dependency:

```php
wp_enqueue_script(
    'my-plugin-block-frontend',
    plugins_url( 'block-frontend.js', __FILE__ ),
    array( 'jquery', 'kdc-qtap-frontend-helpers' ),
    MY_PLUGIN_VERSION,
    true
);
```

### 2. Use it in your JS

```javascript
(function($) {
    'use strict';

    $('#my-pay-button').on('click', function() {
        var loader = window.KdcQtapUI.showPageLoader('Processing payment...');

        $.ajax({
            url: kdcQtapAjax.url,
            method: 'POST',
            data: { action: 'my_plugin_charge', ... },
            success: function(response) {
                loader.setMessage('Redirecting...');
                window.location.href = response.redirect_url;
            },
            error: function(xhr) {
                window.KdcQtapUI.showMessage($('#messages'), 'Payment failed', 'error', false);
            },
            complete: function() {
                loader.hide();   // ALWAYS hide, even on error
            }
        });
    });
})(jQuery);
```

---

## API Reference

### `KdcQtapUI.showPageLoader(message)`

Show the overlay. Returns a handle.

| Param | Type | Description |
|-------|------|-------------|
| `message` | string (optional) | Display text. Falls back to localized "Processing..." if omitted. |

**Returns:**
```javascript
{
    setMessage: function(text) { ... },  // Update the message in-place
    hide: function() { ... }              // Equivalent to KdcQtapUI.hidePageLoader()
}
```

### `KdcQtapUI.hidePageLoader(force)`

Hide the overlay.

| Param | Type | Description |
|-------|------|-------------|
| `force` | boolean (optional) | If `true`, hides regardless of ref count. Use only as cleanup on page unload. |

---

## Ref-Counting Behaviour

`showPageLoader()` is **ref-counted** — multiple calls stack. This lets you nest operations safely:

```javascript
// Outer flow
var outerLoader = KdcQtapUI.showPageLoader('Saving order...');

// During that, fire a sub-operation
var innerLoader = KdcQtapUI.showPageLoader('Verifying with bank...');
// ... bank call completes ...
innerLoader.hide();  // Overlay stays — outer is still active

// Outer finishes
outerLoader.hide();  // NOW the overlay is removed
```

If you find yourself fighting the ref counter, you probably want to call `hidePageLoader(true)` to force-clear before showing a new one. But the pattern above is usually correct.

---

## Common Patterns

### A) Promise / fetch

```javascript
async function submitForm(data) {
    var loader = KdcQtapUI.showPageLoader('Submitting...');
    try {
        const res = await fetch('/wp-json/my/v1/submit', {
            method: 'POST',
            headers: { 'X-WP-Nonce': window.kdcQtapApi.nonce },
            body: JSON.stringify(data)
        });
        const json = await res.json();
        if (!res.ok) throw new Error(json.message || 'Failed');
        loader.setMessage('Saved! Redirecting...');
        window.location.href = json.next_url;
    } catch (e) {
        KdcQtapUI.showMessage($('#messages'), e.message, 'error', false);
    } finally {
        loader.hide();   // CRITICAL: always hide
    }
}
```

### B) Multi-step flow with progress messages

```javascript
function checkout() {
    var loader = KdcQtapUI.showPageLoader('Validating cart...');

    return validateCart()
        .then(function() {
            loader.setMessage('Charging card...');
            return chargeCard();
        })
        .then(function() {
            loader.setMessage('Sending receipt...');
            return sendReceipt();
        })
        .then(function() {
            loader.setMessage('Done! Redirecting...');
            window.location.href = '/thank-you';
        })
        .catch(function(err) {
            KdcQtapUI.showMessage($('#messages'), err.message, 'error', false);
        })
        .finally(function() {
            loader.hide();
        });
}
```

### C) jQuery AJAX

```javascript
function verifyOtp(otp) {
    var loader = KdcQtapUI.showPageLoader('Verifying OTP...');
    $.post(ajaxurl, { action: 'my_verify_otp', otp: otp, _wpnonce: nonce })
        .done(function(res) {
            if (res.success) {
                loader.setMessage('Verified! Redirecting...');
                window.location.reload();
            } else {
                KdcQtapUI.showMessage($('#err'), res.data.message, 'error', false);
            }
        })
        .fail(function() {
            KdcQtapUI.showMessage($('#err'), 'Network error', 'error', false);
        })
        .always(function() {
            loader.hide();
        });
}
```

---

## Critical Rules

1. **Always pair `show` with `hide`.** Use `try/finally` (async/await), `.always()` (jQuery), or `complete:` callbacks. Never let a code path leave the overlay stuck.

2. **Never show without a clear message.** Default "Processing..." is acceptable, but a specific message ("Charging card", "Sending OTP") gives much better UX.

3. **Don't show for fast operations.** If the AJAX call typically returns in <300ms, the overlay flashes annoyingly. For those, prefer `setButtonLoading()`.

4. **Update messages during long operations.** Use `loader.setMessage('Almost done...')` after a few seconds so users know you're alive.

5. **Don't put forms inside the overlay.** It's blocking by design — no inputs visible, no cancel button. If users need to act mid-transaction, build a modal instead.

---

## Accessibility Notes (already handled)

The overlay sets:
- `role="alertdialog"` so screen readers announce it
- `aria-modal="true"` so AT knows the rest is blocked
- `aria-busy="true"` to indicate work-in-progress
- The message has `aria-live="polite"` so updates via `setMessage()` are announced
- Reduced-motion preference (`prefers-reduced-motion`) slows the spinner

You don't need to add any additional ARIA attributes.

---

## Localization

The default "Processing..." message is translatable via the `kdc-qtap` text domain. Override per-call by passing your own message (translated in your child plugin's text domain):

```javascript
KdcQtapUI.showPageLoader(myPluginI18n.processingPayment);
```

Where `myPluginI18n` is a `wp_localize_script` payload from your plugin.

---

## Troubleshooting

**Overlay doesn't appear:**
- Check `window.KdcQtapUI` is defined (means parent helpers loaded). If `undefined`, you didn't enqueue `kdc-qtap-frontend-helpers` as a dependency.
- Check browser console for JS errors before your `showPageLoader()` call.

**Overlay never disappears:**
- You forgot a `hide()`. Use `try/finally`. As an emergency, call `KdcQtapUI.hidePageLoader(true)` from console to force-clear.

**Overlay appears behind other elements:**
- Some third-party plugins use `z-index: 999999999`. Override in your block CSS:
  ```css
  .kdc-qtap-page-overlay { z-index: 1000000000 !important; }
  ```

**Body scroll remains locked after page navigation:**
- Should not happen — overlay is removed before redirect. If it does, call `hidePageLoader(true)` before `window.location.href = ...`.

---

## Minimal Checklist for AI Agents Implementing This

When wiring up a new block transaction in a child plugin:

- [ ] Block PHP enqueues `kdc_qtap_enqueue_frontend_components()` in render
- [ ] Block JS lists `'kdc-qtap-frontend-helpers'` in `wp_enqueue_script` deps
- [ ] Every async action that mutates server state calls `showPageLoader()` first
- [ ] Every `showPageLoader()` is matched by `hide()` in `finally` / `complete` / `always`
- [ ] Messages are specific ("Charging card", not "Loading")
- [ ] Use `loader.setMessage()` for multi-step flows
- [ ] Test failure paths — kill the network mid-request and confirm overlay clears
- [ ] Test on mobile viewport (overlay must be visible above fixed nav bars)

---

**Version:** 1.0.0
**Last updated:** 2026-04-25
**Source:** [kdc-qtap/assets/js/kdc-qtap-frontend-helpers.js](kdc-qtap/assets/js/kdc-qtap-frontend-helpers.js) (`showPageLoader`, `hidePageLoader`)
