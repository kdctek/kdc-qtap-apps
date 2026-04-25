# qTap Notifications Summary Card — Child Plugin Integration Guide

**Parent plugin:** qTap App (`kdc-qtap` v2.7.9+)
**For:** AI assistants and developers building child plugins (mobile, finance, events, kyc, etc.) that send notifications and want to surface stats + cross-references on their own admin pages.

---

## What is the Summary Card?

A drop-in admin UI card you place on YOUR child plugin's settings page. It shows:

- Each notification type your plugin owns (registered via `kdc_qtap_notification_type_owners`)
- 7-day sent / failed counts per type
- "Last sent" timestamp per type with a status dot (green/red/yellow)
- An **Edit Template** button per row — deep-links into wherever your editor lives (parent's Templates section by default; you can override via filter)
- A **View all logs** footer button — deep-links into the parent's Notifications log filtered to your plugin's `source`

This means admins navigating from YOUR settings page can see "is my plugin sending OK?" without leaving for the parent's Notifications tab — and when they DO need to dig deeper, the deep links land them in the right pre-filtered view.

---

## When to Use It

Drop the card on the main settings page of any child plugin that:

- Sends notifications via `kdc_qtap_send_notification()` with a non-empty `source`
- Registers at least one type via the `kdc_qtap_notification_type_owners` filter (REQUIRED — without this the card renders nothing)

If your plugin doesn't currently send any notifications, skip the card.

---

## Quick Start

### Step 1: Register your types

In your child plugin's notifications class (or wherever you set up notification hooks):

```php
add_filter( 'kdc_qtap_notification_type_owners', function( $owners ) {
    // For every notification type your plugin sends, add an entry mapping
    // type_key => your-plugin-slug.
    $owners['my_plugin_event_completed'] = 'my-plugin';
    $owners['my_plugin_event_cancelled'] = 'my-plugin';
    $owners['my_plugin_invitation_sent'] = 'my-plugin';
    return $owners;
} );
```

### Step 2: Drop the card on your admin page

In your settings page render method:

```php
public function render_my_settings_page() {
    ?>
    <div class="wrap">
        <h1><?php esc_html_e( 'My Plugin Settings', 'my-plugin' ); ?></h1>

        <?php
        // ... your settings UI ...

        // The summary card. Renders nothing if no types are registered for this source.
        if ( function_exists( 'kdc_qtap_render_notifications_summary' ) ) {
            kdc_qtap_render_notifications_summary( 'my-plugin' );
        }
        ?>
    </div>
    <?php
}
```

That's it. Admins will see notification stats inline.

---

## API Reference

### `kdc_qtap_render_notifications_summary( $source, $args = array() )`

Render the card. Echoes HTML by default.

| Param        | Type     | Default            | Description |
|--------------|----------|--------------------|-------------|
| `$source`    | string   | (required)         | Your plugin slug — must match what you used in `type_owners` and in `kdc_qtap_send_notification()` calls. |
| `$args.title`| string   | `'Notifications'`  | Card heading. |
| `$args.days` | int      | `7`                | Stat window in days. |
| `$args.echo` | bool     | `true`             | Echo HTML directly (default). Pass `false` to return as a string. |

### `kdc_qtap_get_notifications_admin_url( $args = array() )`

Build a deep-link URL into the parent's Notifications admin tab. Used internally by the card; expose this if you build your own UI.

| Arg        | Type    | Description |
|------------|---------|-------------|
| `section`  | string  | `'logs'` (default), `'templates'`, `'channels'`, `'settings'`. |
| `source`   | string  | Pre-filter logs/templates to this plugin slug. |
| `type`     | string  | Open a specific type's editor (only with `section=templates`). |

```php
// Link to your plugin's logs:
$logs = kdc_qtap_get_notifications_admin_url( array( 'source' => 'my-plugin' ) );

// Link to a specific template:
$tpl = kdc_qtap_get_notifications_admin_url( array(
    'section' => 'templates',
    'type'    => 'my_plugin_event_completed',
) );
```

### `kdc_qtap_get_template_edit_url( $type, $source = '' )`

Build the "Edit Template" URL for a specific type. Routes through the
`kdc_qtap_notification_template_edit_url` filter so plugins with their own
template editors can intercept.

```php
// In your child plugin: route Edit Template clicks to YOUR template editor:
add_filter( 'kdc_qtap_notification_template_edit_url', function( $url, $type, $source ) {
    if ( 'my-plugin' !== $source ) {
        return $url; // Don't touch other plugins' URLs.
    }
    return admin_url( 'admin.php?page=my-plugin&tab=templates&edit=' . $type );
}, 10, 3 );
```

If you don't override the filter, Edit Template links go to the parent's centralized Templates editor (under `qTap App > Notifications > Templates`).

---

## Deep-Link Query Param Contract

The parent's Notifications tab reads these query params:

| Param      | Effect |
|------------|--------|
| `?tab=notifications` | Open the Notifications tab. |
| `?section=templates` | Open the Templates section (when implemented in parent). |
| `?source=my-plugin`  | Pre-filter the logs view to this plugin's notification types. |
| `?type=my_plugin_x`  | Open the specific type's template editor (with `section=templates`). |

All of these are constructed for you by `kdc_qtap_get_notifications_admin_url()` and `kdc_qtap_get_template_edit_url()`. Don't hand-build URLs unless you have a special case.

---

## Required Setup Recap

For the summary card to work, your plugin MUST:

- ✅ Register types via `kdc_qtap_notification_type_owners` filter
- ✅ Pass `source` in EVERY `kdc_qtap_send_notification()` call (matching the slug in `type_owners`)
- ✅ Call `kdc_qtap_render_notifications_summary( 'your-slug' )` on the admin page

Without all three, the card will either render nothing (no types registered) or show zero stats (logs not attributed).

---

## Where Domain-Specific Settings Stay (Reminder Schedules, etc.)

Templates centralize to the parent. **Domain-specific settings stay with the child plugin.**

Example: Finance keeps its **Reminder Schedule** (days-before-due, time-of-day, batch size) on its own admin page. Why: schedule timing is fee-domain business logic — when reminders fire is a child concern; what the reminder *says* is centralized. The parent provides the cron infrastructure; the child decides when to use it.

When designing your child plugin's admin UI:

- Templates → parent (centralized, deep-link via the card)
- Channel enable/disable per type → can stay on your child plugin admin (use a simple settings page)
- Domain timing/scheduling/batch logic → stay with your child plugin
- Variables → register once via `kdc_qtap_register_notification_variables`, parent surfaces them in editor

---

## Migration Note

If your plugin previously stored admin-edited templates in your own option key, the parent runs a one-time migration on upgrade for the `kdc-qtap-finance` plugin specifically (its old key was `kdc_qtap_finance_settings.notification_templates`). For other plugins:

- The parent does NOT auto-migrate from arbitrary plugin options.
- If you have legacy templates to import, do it once during your plugin's upgrade routine using `kdc_qtap_notifications()->save_template( $type, $fields )`.
- After migration, switch your `send_*` methods to drop the legacy lookup — `kdc_qtap_send_notification()` already pulls customized templates from the parent option automatically.

```php
// Example one-time migration in your child plugin:
private function migrate_templates_to_parent_v2() {
    if ( get_option( 'my_plugin_templates_migrated', false ) ) return;
    $legacy = get_option( 'my_plugin_settings', array() );
    if ( ! empty( $legacy['notification_templates'] ) ) {
        foreach ( $legacy['notification_templates'] as $type => $template ) {
            kdc_qtap_notifications()->save_template( $type, $template );
        }
    }
    update_option( 'my_plugin_templates_migrated', true );
}
```

---

## Verification Checklist

After integrating, verify:

- [ ] Card appears on your child plugin admin page with rows for each registered type
- [ ] Sent/failed counts are non-zero after sending a real notification
- [ ] "Last sent" timestamp updates after sends
- [ ] Clicking "Edit template" lands on a usable editor (parent's centralized OR your custom one via filter)
- [ ] Clicking "View all logs" lands on the parent's Notifications tab pre-filtered to your `source`
- [ ] Removing all `type_owners` registrations causes the card to render nothing (graceful empty state)
- [ ] If your plugin has no types yet, calling the helper renders nothing (no error)

---

**Version:** 1.0.0
**Last updated:** 2026-04-26
**Source:** [kdc-qtap/includes/kdc-qtap-notification-functions.php](kdc-qtap/includes/kdc-qtap-notification-functions.php) (`kdc_qtap_render_notifications_summary`, `kdc_qtap_get_template_edit_url`)
