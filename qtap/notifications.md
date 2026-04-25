# qTap Notification System - Child Plugin Integration Guide

**Version:** 2.7.9
**Last Updated:** April 2026

## Overview

The qTap Notification System provides a robust, hook-based notification architecture that allows child plugins to:

1. **Send notifications** through multiple channels (email, SMS, WhatsApp, webhooks, etc.)
2. **Register custom channels** for new delivery methods
3. **Register custom notification types** with default channels and priorities
4. **Register types as ADMIN-EDITABLE** so admins can customize subject/body via parent's centralized Templates editor (v2.7.9+)
5. **Surface a summary card** on the child plugin's admin page with stats + deep links into the parent's Notifications UI (v2.7.9+)
6. **Hook into the notification lifecycle** to modify, log, or extend functionality

## Where Things Live (v2.7.9+)

```
Parent qTap > Notifications              Child Plugin Admin
├── Logs (all sources, source filter)    ├── Settings...
├── Templates (canonical store, all      └── kdc_qtap_render_notifications_summary( source )
│   types from all plugins)                  └── Card with stats per type +
├── Channels                                     deep links to parent
└── Settings
```

**Templates are stored centrally** in the `kdc_qtap_notification_templates` option (parent owns it). The lookup chain when sending: admin-customized template (parent option) → registered defaults (`kdc_qtap_default_notification_templates` filter) → empty.

## Creating an Admin-Editable Notification Type (v2.7.9+)

This is the canonical recipe. Use ALL these hooks together for one notification type:

| Hook / Function                           | Purpose                                          | Required |
|-------------------------------------------|--------------------------------------------------|----------|
| `kdc_qtap_notification_init` (action)     | Hook point for type registration                 | YES |
| `kdc_qtap_register_notification_type()`   | Register type metadata (label, default_channels) | YES |
| `kdc_qtap_default_notification_templates` | Provide default subject/body/whatsapp data       | YES |
| `kdc_qtap_register_notification_variables`| Declare `{{variables}}` your template uses       | If using vars |
| `kdc_qtap_notification_type_owners` ⭐NEW | Declare your plugin owns this type               | YES (for cross-ref) |
| `kdc_qtap_notification_template_edit_url` | (Optional) Override where Edit Template lands    | Optional |
| `kdc_qtap_scheduled_notification_types`   | If your type fires from cron, not user actions   | Only if scheduled |

### Minimum Viable Recipe

```php
class My_Plugin_Notifications {
    public function __construct() {
        add_action( 'kdc_qtap_notification_init',                array( $this, 'register_type' ) );
        add_filter( 'kdc_qtap_default_notification_templates',   array( $this, 'register_defaults' ) );
        add_filter( 'kdc_qtap_register_notification_variables',  array( $this, 'register_variables' ) );
        add_filter( 'kdc_qtap_notification_type_owners',         array( $this, 'register_owners' ) );
    }

    /** Register the type so it appears in the parent's UI. */
    public function register_type() {
        kdc_qtap_register_notification_type( 'my_plugin_event_completed', array(
            'label'            => __( 'Event Completed', 'my-plugin' ),
            'description'      => __( 'Sent when an event completes successfully', 'my-plugin' ),
            'default_channels' => array( 'email', 'whatsapp' ),
            'default_priority' => 'normal',
        ) );
    }

    /** Provide default subject/body. Admin can override via parent's Templates editor. */
    public function register_defaults( $templates ) {
        $templates['my_plugin_event_completed'] = array(
            'subject' => 'Event "{{event_title}}" completed',
            'message' => "Hi {{user_name}},\n\n" .
                         "Your event \"{{event_title}}\" completed at {{event_time:human}}.\n\n" .
                         "View it: {{event_url}}",
        );
        return $templates;
    }

    /** Declare {{variables}} this type uses (auto-surfaces in parent editor's variable palette). */
    public function register_variables( $vars ) {
        $vars['my_plugin'] = array(
            'label'     => __( 'My Plugin', 'my-plugin' ),
            'variables' => array(
                'event_title' => __( 'Event title', 'my-plugin' ),
                'event_time'  => __( 'Event datetime', 'my-plugin' ),
                'event_url'   => __( 'Event public URL', 'my-plugin' ),
                'user_name'   => __( 'Recipient display name', 'my-plugin' ),
            ),
        );
        return $vars;
    }

    /** ⭐ NEW: declare your plugin owns this type. Required for cross-ref UI to work. */
    public function register_owners( $owners ) {
        $owners['my_plugin_event_completed'] = 'my-plugin';
        // Add one entry per type your plugin owns:
        // $owners['my_plugin_event_cancelled'] = 'my-plugin';
        return $owners;
    }

    /** Then anywhere in your plugin, send the notification: */
    public function send_event_completed( $event, $user ) {
        kdc_qtap_send_notification( array(
            'type'      => 'my_plugin_event_completed',
            'recipient' => array( 'email' => $user->user_email, 'user_id' => $user->ID ),
            'channels'  => array( 'email', 'whatsapp' ),
            'data'      => array(
                'event_title' => $event->title,
                'event_time'  => $event->starts_at,
                'event_url'   => get_permalink( $event->post_id ),
                'user_name'   => $user->display_name,
            ),
            'source'    => 'my-plugin', // ALWAYS pass — drives logs filtering and the summary card
        ) );
    }
}
new My_Plugin_Notifications();
```

That's it. The admin can now find this type in **qTap App > Notifications > Templates**, edit subject/body/WhatsApp template, and your `send_event_completed()` will use the customized version automatically (lookup falls back to the default if untouched).

### Show the Summary Card on Your Plugin's Admin Page

```php
public function render_my_settings_page() {
    ?>
    <div class="wrap">
        <h1>My Plugin Settings</h1>
        <?php
        // ... your settings UI ...

        // Drop the notifications summary card. Renders nothing if you haven't
        // registered any types via kdc_qtap_notification_type_owners filter.
        if ( function_exists( 'kdc_qtap_render_notifications_summary' ) ) {
            kdc_qtap_render_notifications_summary( 'my-plugin' );
        }
        ?>
    </div>
    <?php
}
```

The card lists each registered type with 7-day sent/failed/latest stats, an "Edit template" button per row, and a "View all logs" footer link.

### Override the Edit Template URL (optional)

If your plugin already has its own template editor (e.g., kdc-qtap-finance has one), point Edit Template at it instead of the parent:

```php
add_filter( 'kdc_qtap_notification_template_edit_url', function( $url, $type, $source ) {
    if ( 'my-plugin' !== $source ) {
        return $url;
    }
    return admin_url( 'admin.php?page=my-plugin&tab=templates&edit=' . $type );
}, 10, 3 );
```

## What NOT To Do

- ❌ **DO NOT build your own template editor** for new plugins. The parent will provide a centralized editor (lifted from finance) — leverage it. The summary card already deep-links into it.
- ❌ **DO NOT save admin-edited templates to your own option key.** Storage is `kdc_qtap_notification_templates` (parent option). The parent runs a one-time migration on upgrade for any existing self-managed templates.
- ❌ **DO NOT duplicate the variable set** — register once via `kdc_qtap_register_notification_variables` and the parent surfaces them in the editor.
- ❌ **DO NOT forget to pass `source`** in `kdc_qtap_send_notification()` calls — without it, the summary card and Source filter can't attribute the log row to your plugin.
- ❌ **DO NOT skip `kdc_qtap_notification_type_owners`** — without it, the summary card renders nothing and Edit Template links can't be routed.

## Architecture

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Child Plugin                                 │
│                                                                  │
│  kdc_qtap_send_notification([                                   │
│      'recipient' => 'user@example.com',                         │
│      'subject'   => 'Order Shipped',                            │
│      'message'   => 'Your order has shipped!',                  │
│      'type'      => 'order_update',                             │
│      'channels'  => ['email', 'sms'],                           │
│  ]);                                                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   qTap Notification System                       │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Filters   │  │  Dispatcher │  │   Logging   │              │
│  │  & Hooks    │──│             │──│             │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │  Email   │   │   SMS    │   │ Webhook  │
        │ Channel  │   │ Channel  │   │ Channel  │
        └──────────┘   └──────────┘   └──────────┘
```

## Quick Start

### Sending a Simple Notification

```php
// Send a basic email notification
kdc_qtap_send_notification( array(
    'recipient' => get_option( 'admin_email' ),
    'subject'   => 'New Order Received',
    'message'   => 'You have received a new order #12345.',
) );
```

### Sending a Multi-Channel Notification

```php
kdc_qtap_send_notification( array(
    'recipient' => array(
        'email'   => 'customer@example.com',
        'phone'   => '+1234567890',
        'user_id' => 123,
    ),
    'subject'   => 'Order Shipped',
    'message'   => 'Your order #{{order_id}} has been shipped. Tracking: {{tracking}}',
    'type'      => 'order_update',
    'channels'  => array( 'email', 'sms', 'webhook' ),
    'priority'  => 'high',
    'data'      => array(
        'order_id' => 12345,
        'tracking' => 'ABC123XYZ',
    ),
    'source'    => 'my-child-plugin',
) );
```

### WhatsApp with Multiple Recipients (v2.2.25+)

Send the same WhatsApp message to multiple phone numbers at once:

```php
kdc_qtap_send_notification( array(
    'recipient' => array(
        'phone' => array(
            '+919876543210',
            '+919876543211',
            '+919876543212',
        ),
    ),
    'subject'  => 'Reminder',
    'message'  => "template=appointment_reminder\nbody1=Tomorrow\nbody2=10:00 AM",
    'channels' => array( 'whatsapp' ),
) );
```

## Variable Replacement Utilities (v2.3.14+)

The parent plugin provides **global utility functions** for variable replacement. These are NOT tied to notifications or any channel - use them anywhere in your child plugin.

### Core Functions

#### `kdc_qtap_replace_variables( $string, $data )`

Replace `{{variables}}` in any string. Supports format modifiers.

```php
// Basic usage
$message = kdc_qtap_replace_variables(
    'Hello {{name}}, your payment of {{amount}} is due on {{due_date}}.',
    array(
        'name'     => 'John Doe',
        'amount'   => 108050,
        'due_date' => '2026-01-15',
    )
);
// Result: "Hello John Doe, your payment of ₹ 1,08,050 is due on January 15, 2026."

// With format modifiers
$message = kdc_qtap_replace_variables(
    'Pay {{amount:value}} by {{due_date:human}}.',
    array(
        'amount'   => 108050,
        'due_date' => '2026-01-15',
    )
);
// Result: "Pay 108050 by in 5 days."
```

#### `kdc_qtap_replace_variables_in_array( $array, $data )`

Replace variables in all strings within an array (recursively).

```php
$processed = kdc_qtap_replace_variables_in_array(
    array(
        'subject' => 'Invoice #{{invoice_id}}',
        'items'   => array(
            'Fee: {{amount:lakh}}',
            'Due: {{due_date:short}}',
        ),
    ),
    array(
        'invoice_id' => 12345,
        'amount'     => 108050,
        'due_date'   => '2026-01-15',
    )
);
// Result: ['subject' => 'Invoice #12345', 'items' => ['Fee: ₹ 1,08,050', 'Due: Jan 15, 2026']]
```

### Use In Child Plugin (Recommended Approach)

**Process your data BEFORE sending notifications:**

```php
// 1. Prepare your raw data
$data = array(
    'student_name' => $student->display_name,
    'amount'       => $fee->amount,           // Raw number: 108050
    'due_date'     => $fee->due_date,         // ISO date: 2026-01-15
    'payment_url'  => $payment_link,
);

// 2. Process your templates using parent utility
$whatsapp_body = kdc_qtap_replace_variables_in_array(
    array(
        '{{student_name}}',
        '{{amount:value}}',
        '{{due_date:Y-m-d}}',
    ),
    $data
);
// Result: ['John Doe', '108050', '2026-01-15']

$email_message = kdc_qtap_replace_variables(
    'Hello {{student_name}}, please pay {{amount:lakh}} by {{due_date:human}}.',
    $data
);
// Result: 'Hello John Doe, please pay ₹ 1,08,050 by in 5 days.'

// 3. Send notification with processed data
kdc_qtap_send_notification( array(
    'recipient' => $student->phone,
    'message'   => $email_message,
    'whatsapp_data' => array(
        'template' => 'payment_reminder|en',
        'body'     => $whatsapp_body,         // Already processed!
        'buttons'  => array( 'url|' . $data['payment_url'] ),
    ),
    'data'     => $data,  // Store for logging
    'channels' => array( 'email', 'whatsapp' ),
) );
```

### Built-in Variables

These are always available without adding to `data`. All variables support [format modifiers](#format-modifiers-v2314).

| Variable | Description | Example Formats |
|----------|-------------|-----------------|
| `{{site_name}}` | WordPress site name | `:upper`, `:lower`, `:title` |
| `{{site_url}}` | Site home URL | `:domain`, `:path` |
| `{{site_icon}}` | Site favicon/icon URL (v2.3.18+) | `:suffix` |
| `{{site_logo}}` | Site logo URL (v2.3.18+) | `:suffix` |
| `{{admin_email}}` | Admin email address | `:upper`, `:lower` |
| `{{current_year}}` | Current year (e.g., 2026) | `:value` |

**Examples with Format Modifiers:**
```php
'{{site_name:upper}}'        // "MY SCHOOL NAME"
'{{site_name:title}}'        // "My School Name"
'{{site_url:domain}}'        // "example.com"
'{{admin_email:lower}}'      // "admin@example.com"
```

**Site Logo Detection (v2.3.18+):**

The `{{site_logo}}` variable automatically detects logos from:
1. Custom Logo (Customizer → Site Identity)
2. Block theme Site Logo block
3. Theme-specific logo settings

**Fallback (v2.3.23+):** If no site icon or logo is found, the plugin's default qTap logo is used.

**Customizing Site Assets:**

```php
// Override site icon URL
add_filter( 'kdc_qtap_site_icon_url', function( $url ) {
    return 'https://example.com/custom-icon.png';
} );

// Override site logo URL
add_filter( 'kdc_qtap_site_logo_url', function( $url ) {
    return 'https://example.com/custom-logo.png';
} );
```

### Format Modifiers (v2.3.14+)

Variables support format modifiers using the syntax `{{variable:format}}`:

#### Amount/Number Formats

Pass **raw numeric values** in data, and let the parent handle formatting:

```php
'data' => array(
    'amount_due' => 108050,  // Raw number, NOT pre-formatted
)
```

| Syntax | Output | Description |
|--------|--------|-------------|
| `{{amount_due}}` | ₹ 1,08,050 | Default: Indian format with currency |
| `{{amount_due:value}}` | 108050 | Raw numeric value |
| `{{amount_due:lakh}}` | ₹ 1,08,050 | Indian lakh format |
| `{{amount_due:international}}` | ₹ 108,050 | International comma format |
| `{{amount_due:words}}` | One Lakh Eight Thousand Fifty | Amount in words |
| `{{amount_due:compact}}` | ₹ 1.08 L | Compact format (K/L/Cr) |
| `{{amount_due:decimal}}` | ₹ 108,050.00 | With 2 decimal places |

#### Date Formats

Pass dates as **timestamps, ISO strings, or any parseable format**:

```php
'data' => array(
    'due_date' => '2026-01-15',  // ISO format (recommended)
    // OR: 'due_date' => strtotime( '2026-01-15' ),  // Timestamp
    // OR: 'due_date' => '15 January 2026',          // Human readable
)
```

| Syntax | Output | Description |
|--------|--------|-------------|
| `{{due_date}}` | January 15, 2026 | WordPress date format setting |
| `{{due_date:human}}` | in 5 days | Human readable (X ago / in X) |
| `{{due_date:relative}}` | Tomorrow | Today/Tomorrow/Yesterday/in X days |
| `{{due_date:Y-m-d}}` | 2026-01-15 | PHP date format |
| `{{due_date:d-m-Y}}` | 15-01-2026 | Day-Month-Year |
| `{{due_date:short}}` | Jan 15, 2026 | Short format |
| `{{due_date:long}}` | January 15, 2026 | Long format |
| `{{due_date:full}}` | Thursday, January 15, 2026 | Full with weekday |
| `{{due_date:time}}` | 10:30 AM | Time only |
| `{{due_date:datetime}}` | January 15, 2026 10:30 AM | Date and time |
| `{{due_date:iso}}` | 2026-01-15T00:00:00+05:30 | ISO 8601 format |

#### String Formats

| Syntax | Output | Description |
|--------|--------|-------------|
| `{{name:upper}}` | JOHN DOE | Uppercase |
| `{{name:lower}}` | john doe | Lowercase |
| `{{name:title}}` | John Doe | Title Case |
| `{{name:ucfirst}}` | John doe | First letter uppercase |

#### URL Formats (v2.3.16+)

Strip site URL or extract URL parts:

```php
'data' => array(
    'payment_url' => 'https://example.com/payment/invoice/123?token=abc',
)
```

| Syntax | Output | Description |
|--------|--------|-------------|
| `{{payment_url}}` | https://example.com/payment/invoice/123?token=abc | Full URL |
| `{{payment_url:suffix}}` | /payment/invoice/123?token=abc | Path only (strips site URL) |
| `{{payment_url:path}}` | /payment/invoice/123?token=abc | Same as suffix |
| `{{payment_url:domain}}` | example.com | Domain/host only |
| `{{payment_url:query}}` | token=abc | Query string only (without ?) |

**Use case:** WhatsApp buttons often need just the path suffix:
```php
'whatsapp_data' => array(
    'buttons' => array( 'url|{{payment_url:suffix}}' ),
),
```

#### Example Usage

```php
kdc_qtap_send_notification( array(
    'recipient' => $student_phone,
    'message'   => 'Hi {{student_name:title}}, your fee of {{amount:lakh}} is due {{due_date:relative}}.',
    'whatsapp_data' => array(
        'template' => 'fee_reminder|en',
        'body'     => array(
            '{{student_name}}',
            '{{amount:value}}',        // WhatsApp might need raw value
            '{{due_date:Y-m-d}}',      // Or specific format
        ),
    ),
    'data' => array(
        'student_name' => 'john doe',
        'amount'       => 108050,      // Raw number
        'due_date'     => '2026-01-15', // ISO date
    ),
) );
```

#### Customizing Currency Symbol

```php
// Change currency symbol globally
add_filter( 'kdc_qtap_currency_symbol', function( $symbol ) {
    return '$';  // Use dollar instead of rupee
} );
```

The notification system will send to each number individually and return aggregated results:

```php
array(
    'success'          => true,
    'total_recipients' => 3,
    'sent'             => 3,
    'failed'           => 0,
    'results'          => array(
        '919876543210' => array( 'success' => true, ... ),
        '919876543211' => array( 'success' => true, ... ),
        '919876543212' => array( 'success' => true, ... ),
    ),
    'response_message' => 'Sent to 3 of 3 recipients.',
)
```

### WhatsApp Template Language Code (v2.2.27+)

Specify the template language code using the `template_name|language_code` format:

```php
kdc_qtap_send_notification( array(
    'recipient'          => array( 'phone' => '+919876543210' ),
    'subject'            => 'Order Confirmation',
    'whatsapp_template'  => 'order_confirmation|en_US',  // template_name|language_code
    'whatsapp_body'      => "Order #12345\n₹1,500.00",
    'channels'           => array( 'whatsapp' ),
) );
```

**Supported formats:**

| Format | Example | Result |
|--------|---------|--------|
| `template_name` | `order_confirmation` | Template: `order_confirmation`, Language: `en` (default) |
| `template_name\|language` | `order_confirmation\|en_US` | Template: `order_confirmation`, Language: `en_US` |
| `template_name\|language` | `order_confirmation\|hi` | Template: `order_confirmation`, Language: `hi` |

**Alternative methods:**

1. Using `whatsapp_data` array:
```php
kdc_qtap_send_notification( array(
    'recipient' => array( 'phone' => '+919876543210' ),
    'whatsapp_data' => array(
        'template' => 'order_confirmation',
        'language' => 'en_US',
        'body'     => array( 'Order #12345', '₹1,500.00' ),
    ),
    'channels' => array( 'whatsapp' ),
) );
```

2. Using legacy message format:
```php
kdc_qtap_send_notification( array(
    'recipient' => array( 'phone' => '+919876543210' ),
    'message'   => "template=order_confirmation|en_US\nbody1=Order #12345\nbody2=₹1,500.00",
    'channels'  => array( 'whatsapp' ),
) );

// Or with separate language key:
'message' => "template=order_confirmation\nlanguage=en_US\nbody1=Order #12345"
```

### WhatsApp Button Components (v2.2.31+)

WhatsApp templates can include dynamic buttons. Use the `type|value` format (pipe separator) to specify button types.

#### Template Format (Required)

```
template_name|language_code
```

- **template_name**: alphanumeric and underscore (`_`) only
- **language_code**: required (e.g., `en`, `en_US`, `hi`, `mr`)

```php
'whatsapp_template' => 'order_confirmation|en'
'whatsapp_template' => 'payment_reminder|en_US'
```

#### Button Format (Required)

```
type|value
```

Each button on a new line. The pipe (`|`) separates the type from the value.

#### Button Type Reference

| Type | Format | Description |
|------|--------|-------------|
| `reply` | `reply\|payload` | Quick reply button with payload |
| `url` | `url\|https://...` | URL button (dynamic suffix) |
| `phone` | `phone\|+919876543210` | Phone number button (dynamic) |
| `call` | `call` | Call permission request button (no value) |
| `code` | `code\|DISCOUNT20` | Copy code/coupon button |
| `flow` | `flow\|token\|{"key":"value"}` | WhatsApp Flow trigger |

**Note:** All buttons must use the `type|value` format. Invalid formats will be rejected with validation errors.

#### Form Field Helpers (v2.3.3+)

**Always use parent plugin helpers** in child plugin notification forms for consistent labels and descriptions across all child plugins:

```php
<table class="form-table">
    <?php kdc_qtap_render_whatsapp_template_field( $saved_template ); ?>
    <?php kdc_qtap_render_whatsapp_header_field( $saved_header ); ?>
    <?php kdc_qtap_render_whatsapp_body_field( $saved_body ); ?>
    <?php kdc_qtap_render_whatsapp_footer_field( $saved_footer ); ?>
    <?php kdc_qtap_render_whatsapp_buttons_field( $saved_buttons ); ?>
</table>
```

**Available form field helpers:**

| Function | Field Type | Description |
|----------|------------|-------------|
| `kdc_qtap_render_whatsapp_template_field()` | Text input | Template name with language code |
| `kdc_qtap_render_whatsapp_header_field()` | Text input | Header variable (text/image/document/video URL) |
| `kdc_qtap_render_whatsapp_body_field()` | Textarea | Body variables (one per line) |
| `kdc_qtap_render_whatsapp_footer_field()` | Text input | Footer variable (60 chars max) |
| `kdc_qtap_render_whatsapp_buttons_field()` | Textarea | Button variables with supported types |

**Customization options:**

```php
<?php kdc_qtap_render_whatsapp_template_field( $value, array(
    'name'     => 'my_notification_template',  // Custom field name
    'id'       => 'my-template-field',         // Custom ID (defaults to name)
    'required' => true,                        // Add required attribute
    'echo'     => false,                       // Return HTML instead of echo
) ); ?>

<?php kdc_qtap_render_whatsapp_buttons_field( $value, array(
    'name'     => 'my_notification_buttons',
    'rows'     => 6,                           // Textarea rows (default: 4)
    'required' => false,
) ); ?>
```

**Output example (Template field):**
```html
<tr>
    <th scope="row">
        <label for="whatsapp_template">Template</label>
    </th>
    <td>
        <input type="text" id="whatsapp_template" name="whatsapp_template" 
               class="regular-text" placeholder="order_confirmation|en"
               pattern="[a-zA-Z0-9_]+\|[a-zA-Z_]+" />
        <p class="description">
            Format: template_name|language_code (e.g., hello_world|en, payment_reminder|en_US)
        </p>
    </td>
</tr>
```

**Output example (Buttons field):**
```html
<tr>
    <th scope="row">
        <label for="whatsapp_buttons">Button Variables</label>
    </th>
    <td>
        <textarea id="whatsapp_buttons" name="whatsapp_buttons" rows="4" 
                  class="large-text code"
                  placeholder="url|https://example.com&#10;reply|confirm&#10;call"></textarea>
        <p class="description">
            One button per line. Format: type|value<br>
            <code>reply|confirm_order</code> — Quick reply button<br>
            <code>url|https://example.com/track</code> — URL button<br>
            <code>phone|+919876543210</code> — Phone number button<br>
            <code>call</code> — Voice call permission (no value needed)<br>
            <code>code|SAVE20</code> — Copy code/coupon button<br>
            <code>flow|flow_token</code> — WhatsApp Flow trigger<br>
            <code>order|payment_id</code> — WhatsApp Pay order button
        </p>
    </td>
</tr>
```

#### Basic Example

```php
kdc_qtap_send_notification( array(
    'recipient'          => array( 'phone' => '+919876543210' ),
    'whatsapp_template'  => 'order_tracking|en',
    'whatsapp_body'      => "Order #12345\nShipped",
    'whatsapp_buttons'   => "url|https://track.example.com/12345\nreply|confirm_received",
    'channels'           => array( 'whatsapp' ),
) );
```

#### All Button Types Example

```php
kdc_qtap_send_notification( array(
    'recipient'          => array( 'phone' => '+919876543210' ),
    'whatsapp_template'  => 'promo_template|en',
    'whatsapp_body'      => "Special offer just for you!",
    'whatsapp_buttons'   => "url|https://shop.example.com/offer/123\ncode|SAVE20\nreply|not_interested",
    'channels'           => array( 'whatsapp' ),
) );
```

#### Generated Payload Examples

**URL Button (`url|`):**
```json
{
  "type": "button",
  "sub_type": "url",
  "index": "0",
  "parameters": [
    { "type": "text", "text": "https://track.example.com/12345" }
  ]
}
```

**Quick Reply Button (`reply|`):**
```json
{
  "type": "button",
  "sub_type": "quick_reply",
  "index": "1",
  "parameters": [
    { "type": "payload", "payload": "confirm_received" }
  ]
}
```

**Copy Code Button (`code|`):**
```json
{
  "type": "button",
  "sub_type": "copy_code",
  "index": "2",
  "parameters": [
    { "type": "coupon_code", "coupon_code": "SAVE20" }
  ]
}
```

**Phone Button (`phone|`):**
```json
{
  "type": "button",
  "sub_type": "url",
  "index": "2",
  "parameters": [
    { "type": "text", "text": "+919876543210" }
  ]
}
```

**Call Permission Button (`call`):**
```json
{
  "type": "button",
  "sub_type": "voice_call",
  "index": "3",
  "parameters": []
}
```

**Flow Button (`flow|token|data`):**
```json
{
  "type": "button",
  "sub_type": "flow",
  "index": "4",
  "parameters": [
    {
      "type": "action",
      "action": {
        "flow_token": "token123",
        "flow_action_data": { "key": "value" }
      }
    }
  ]
}
```

**Order Button (`order|payment_id`) - WhatsApp Pay (v2.3.17+):**

The order button enables WhatsApp Pay integration for in-chat payments. It requires:
1. Payment Provider and Payment Configuration set in qTap → Channels → WhatsApp settings
2. Child plugin to provide order data via the `kdc_qtap_whatsapp_order_data` filter

**Child Plugin Usage:**

```php
// 1. Hook into the order data filter
add_filter( 'kdc_qtap_whatsapp_order_data', function( $order_data, $payment_id, $data, $notification ) {
    // Fetch order details based on payment_id
    $fee = get_fee_by_payment_id( $payment_id );
    
    if ( ! $fee ) {
        return $order_data;
    }
    
    return array(
        'payment_id'       => $payment_id,
        'user_id'          => $fee->user_id,
        // 'recipient_number' is auto-populated from message recipient (v2.3.21+)
        'amount'           => $fee->balance,        // Amount in rupees (NOT paise)
        'fee_name'         => $fee->category_name,  // Max 60 characters (auto-truncated)
        'fee_term'         => $fee->term,
        'academic_year'    => $fee->academic_year,
        'grade'            => $fee->grade,
        'division'         => $fee->division,
    );
}, 10, 4 );

// 2. Send notification with order button
kdc_qtap_send_notification( array(
    'recipient'     => $user->phone,  // This becomes recipient_number automatically
    'whatsapp_data' => array(
        'template' => 'fee_payment|en',
        'body'     => array( $user_name, $amount, $due_date ),
        'buttons'  => array( 'order|' . $payment_id ),  // Order button
    ),
    'data' => array(
        'payment_id' => $payment_id,
        'user_id'    => $user->user_id,
    ),
    'channels' => array( 'whatsapp' ),
) );
```

**Notes:**
- **v2.3.21+:** The `recipient_number` field is automatically populated from the phone number the message is being sent to.
- **v2.3.27+:** The `fee_name` field is automatically truncated to 60 characters (WhatsApp API limit).

**Generated Order Button Payload:**
```json
{
  "type": "button",
  "sub_type": "order_details",
  "index": "0",
  "parameters": [
    {
      "type": "action",
      "action": {
        "order_details": {
          "reference_id": "fee-pay-12345-67890_1736520000",
          "type": "digital-goods",
          "payment_settings": [
            {
              "type": "payment_gateway",
              "payment_gateway": {
                "type": "razorpay",
                "configuration_name": "school_payments",
                "razorpay": {
                  "receipt": "fee-pay-12345-67890_1736520000",
                  "notes": {
                    "recipient_number": "+919876543210",
                    "user_id": "67890",
                    "payment_id": "12345"
                  }
                }
              }
            }
          ],
          "currency": "INR",
          "total_amount": { "value": 1080500, "offset": 100 },
          "order": {
            "status": "pending",
            "items": [
              {
                "name": "Tuition Fee",
                "amount": { "value": 1080500, "offset": 100 },
                "quantity": 1
              }
            ],
            "subtotal": { "value": 1080500, "offset": 100 }
          }
        }
      }
    }
  ]
}
```

**Gateway-Specific Fields:**

| Gateway | Fields |
|---------|--------|
| Razorpay | `receipt`, `notes` (recipient_number, user_id, payment_id, academic_year, fee_category, grade_division) |
| PayU | `udf1` (reference), `udf2` (phone), `udf3` (fee_term), `udf4` (academic_year_grade-division) |
| Billdesk | `additional_info1-7` (reference, phone, user_id, payment_id, academic_year, fee_term, grade-division) |
| Zaakpay | `extra1` (reference), `extra2` (phone) |

**Filters:**

| Filter | Purpose |
|--------|---------|
| `kdc_qtap_whatsapp_order_data` | Provide order details based on payment_id |
| `kdc_qtap_whatsapp_order_action` | Customize the complete order action payload |
| `kdc_qtap_whatsapp_payment_gateway_config` | Customize gateway-specific configuration |

**Actions:**

| Action | Purpose |
|--------|---------|
| `kdc_qtap_whatsapp_order_sent` | Fired after successful order message send (v2.3.28+) |

#### Order Message Tracking (v2.3.28+)

When a WhatsApp order message is sent successfully, the parent plugin fires the `kdc_qtap_whatsapp_order_sent` action. This enables child plugins to:

1. **Track multiple messages per payment** - Each message has a unique `reference_id` with timestamp
2. **Send reminders as replies** - Use the `message_id` (wamid) to thread replies
3. **Map gateway callbacks** - Parse `reference_id` to identify payment records

**Reference ID Format:**
```
fee-pay-{payment_id}-{user_id}_{timestamp}
```
Example: `fee-pay-123-456_1736520000`

**Tracking Data Structure:**

```php
add_action( 'kdc_qtap_whatsapp_order_sent', function( $tracking_data ) {
    // $tracking_data contains:
    // - payment_id: Payment record ID
    // - user_id: User ID
    // - recipient_number: Phone number message was sent to
    // - reference_id: Unique reference with timestamp (e.g., fee-pay-123-456_1736520000)
    // - timestamp: Unix timestamp when message was sent
    // - amount: Payment amount in base currency
    // - fee_name: Fee/item name (truncated to 60 chars)
    // - message_id: WhatsApp message ID (wamid.*) for reply threading
    
    // Store in your tracking table
    global $wpdb;
    $wpdb->insert(
        $wpdb->prefix . 'my_whatsapp_order_tracking',
        array(
            'payment_id'       => $tracking_data['payment_id'],
            'recipient_number' => $tracking_data['recipient_number'],
            'reference_id'     => $tracking_data['reference_id'],
            'message_id'       => $tracking_data['message_id'],
            'sent_at'          => current_time( 'mysql' ),
        ),
        array( '%s', '%s', '%s', '%s', '%s' )
    );
} );
```

**Sending Reminder as Reply:**

To send a reminder as a reply to the original order message, look up the stored `message_id` and pass it in the notification:

```php
// Get the last message ID for this payment
$last_message = $wpdb->get_row( $wpdb->prepare(
    "SELECT message_id FROM {$wpdb->prefix}my_whatsapp_order_tracking 
     WHERE payment_id = %s 
     ORDER BY sent_at DESC LIMIT 1",
    $payment_id
) );

// Send reminder as reply
kdc_qtap_send_notification( array(
    'recipient'     => $user->phone,
    'whatsapp_data' => array(
        'template'   => 'payment_reminder|en',
        'body'       => array( $user_name, $amount, $due_date ),
        'buttons'    => array( 'order|' . $payment_id ),
        'context'    => array(
            'message_id' => $last_message->message_id,  // Reply to this message
        ),
    ),
    'channels' => array( 'whatsapp' ),
) );
```

#### Button Input Methods

| Method | Example |
|--------|---------|
| `whatsapp_buttons` field | `"url\|https://example.com\nreply\|payload\ncode\|SAVE20"` (one per line) |
| `whatsapp_data` array | `'buttons' => array('url\|https://example.com', 'reply\|payload')` |
| Legacy message format | `button0=url\|https://example.com\nbutton1=reply\|payload` |

### Auto-Webhook Behavior (v2.2.25+)

If the webhook channel is enabled in qTap settings, it will **automatically** be included in all notifications, even if your child plugin doesn't specify it in the `channels` array.

This means child plugins don't need to know about or manage webhook configuration. The parent plugin handles it transparently:

```php
// Child plugin sends to email only
kdc_qtap_send_notification( array(
    'recipient' => 'user@example.com',
    'subject'   => 'Order Update',
    'message'   => 'Your order shipped!',
    'channels'  => array( 'email' ),  // Only email specified
) );

// If webhook is enabled in qTap settings, the notification
// will be sent to BOTH email AND webhook automatically.
// The webhook receives all channel results in its payload.
```

The webhook payload includes results from all other channels:

```json
{
  "subject": "Order Update",
  "message": "Your order shipped!",
  "type": "system",
  "channel_results": {
    "email": {
      "success": true,
      "request": { "to": "user@example.com", "subject": "Order Update" },
      "response": { "success": true, "response_message": "Email queued..." }
    }
  }
}
```

## API Reference

### Helper Functions

#### `kdc_qtap_send_notification( $args )`

Send a notification through one or more channels.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `recipient` | mixed | Yes | Email, phone, user ID, or array with multiple |
| `subject` | string | Yes* | Notification subject/title |
| `message` | string | Yes* | Notification message body |
| `type` | string | No | Notification type. Default: `'system'` |
| `channels` | array | No | Channels to use. Default: from type config |
| `priority` | string | No | `'low'`, `'normal'`, `'high'`, `'urgent'`. Default: from type |
| `data` | array | No | Additional data for templates and logging |
| `template` | string | No | Template ID for message formatting |
| `source` | string | No | Source identifier (your plugin slug) |

*At least one of `subject` or `message` is required.

**Returns:** `array` - Unified response structure

```php
$result = kdc_qtap_send_notification( $args );
```

## Response Format (v2.2.30+)

All channels return a **unified response structure** for consistent handling. This structure is identical whether you're sending to one recipient or multiple.

### Main Response Structure

```php
array(
    'success'   => true,                    // Overall success (true if ANY channel succeeded)
    'channels'  => array(
        'email'    => true,                 // Per-channel success boolean
        'whatsapp' => true,
        'sms'      => false,
        'webhook'  => true,
    ),
    'errors'    => array(
        'sms' => 'Channel is disabled.',    // Per-channel error messages (only for failed channels)
    ),
    'responses' => array(                   // Full response data from each channel
        'email'    => array( ... ),
        'whatsapp' => array( ... ),
        'sms'      => array( ... ),
        'webhook'  => array( ... ),
    ),
)
```

### Unified Channel Response Structure

Every channel (email, SMS, WhatsApp, webhook) returns the same structure:

```php
'responses' => array(
    'channel_name' => array(
        // Status counts
        'success'          => true,              // Overall success for this channel
        'total_recipients' => 1,                 // Total recipients processed
        'sent'             => 1,                 // Successfully sent count
        'failed'           => 0,                 // Failed count
        
        // Per-recipient results (keyed by recipient identifier)
        'recipients'       => array(
            'recipient_id' => array(             // Email address, phone number, or URL
                'success'          => true,
                'error'            => null,      // Error message if failed
                'request'          => array(),   // Request data sent
                'response_code'    => 200,       // HTTP response code (if applicable)
                'response_body'    => '...',     // Response body (if applicable)
                'response_message' => 'OK',      // Response message
            ),
        ),
        
        // Top-level logging data
        'request'          => array(             // First/primary request for logging
            'url'     => 'https://api.example.com/send',
            'method'  => 'POST',
            'headers' => array( 'Content-Type', 'Authorization' ),
            'payload' => array( ... ),
        ),
        'response_code'    => 200,               // Last response code
        'response_body'    => '...',             // Last response body
        'response_message' => 'OK',              // Summary message
        'error'            => null,              // Top-level error (if complete failure)
    ),
)
```

### Example: WhatsApp Response (Single Recipient)

```php
'responses' => array(
    'whatsapp' => array(
        'success'          => true,
        'total_recipients' => 1,
        'sent'             => 1,
        'failed'           => 0,
        'recipients'       => array(
            '919876543210' => array(
                'success'          => true,
                'error'            => null,
                'request'          => array(
                    'url'     => 'https://graph.facebook.com/v21.0/123456789/messages',
                    'method'  => 'POST',
                    'headers' => array( 'Content-Type', 'Authorization' ),
                    'payload' => array(
                        'messaging_product' => 'whatsapp',
                        'to'                => '919876543210',
                        'type'              => 'template',
                        'template'          => array( ... ),
                    ),
                ),
                'response_code'    => 200,
                'response_body'    => '{"messaging_product":"whatsapp",...}',
                'response_message' => 'OK',
            ),
        ),
        'request'          => array( ... ),
        'response_code'    => 200,
        'response_body'    => '{"messaging_product":"whatsapp",...}',
        'response_message' => 'OK',
    ),
)
```

### Example: WhatsApp Response (Multiple Recipients)

```php
'responses' => array(
    'whatsapp' => array(
        'success'          => true,
        'total_recipients' => 3,
        'sent'             => 2,
        'failed'           => 1,
        'recipients'       => array(
            '919876543210' => array(
                'success'          => true,
                'response_code'    => 200,
                'response_message' => 'OK',
                // ... full data
            ),
            '919876543211' => array(
                'success'          => true,
                'response_code'    => 200,
                'response_message' => 'OK',
                // ... full data
            ),
            '919876543212' => array(
                'success'          => false,
                'error'            => 'Invalid phone number format',
                // ... full data
            ),
        ),
        'request'          => array( ... ),      // First request
        'response_code'    => 200,               // Last response code
        'response_body'    => '...',             // Last response body
        'response_message' => 'Sent to 2 of 3 recipients.',
    ),
)
```

### Example: Email Response

```php
'responses' => array(
    'email' => array(
        'success'          => true,
        'total_recipients' => 1,
        'sent'             => 1,
        'failed'           => 0,
        'recipients'       => array(
            'user@example.com' => array(
                'success'          => true,
                'error'            => null,
                'response_message' => 'Email queued for delivery to user@example.com',
            ),
        ),
        'request'          => array(
            'to'      => 'user@example.com',
            'subject' => 'Order Update',
            'headers' => array( 'From: ...', 'Content-Type: ...' ),
        ),
        'response_code'    => null,
        'response_body'    => null,
        'response_message' => 'Email queued for delivery to user@example.com',
    ),
)
```

### Error Response (Validation Failure)

If validation fails before any channel is processed:

```php
array(
    'success' => false,
    'error'   => 'Notification recipient is required.',
    'code'    => 'missing_recipient',
)
```

### Handling Responses in Child Plugins

```php
$result = kdc_qtap_send_notification( array(
    'recipient'         => array( 'phone' => '+919876543210' ),
    'whatsapp_template' => 'order_status|en',
    'whatsapp_body'     => "Order #12345\nShipped",
    'channels'          => array( 'whatsapp' ),
) );

// Check overall success
if ( $result['success'] ) {
    // At least one channel succeeded
}

// Check specific channel
if ( ! empty( $result['channels']['whatsapp'] ) ) {
    $whatsapp = $result['responses']['whatsapp'];
    
    // Check counts
    echo "Sent: {$whatsapp['sent']} / {$whatsapp['total_recipients']}";
    
    // Iterate recipients
    foreach ( $whatsapp['recipients'] as $phone => $data ) {
        if ( $data['success'] ) {
            echo "✓ Sent to {$phone}";
        } else {
            echo "✗ Failed for {$phone}: " . ( $data['error'] ?? 'Unknown' );
        }
    }
}

// Handle channel errors
if ( ! empty( $result['errors'] ) ) {
    foreach ( $result['errors'] as $channel => $error ) {
        error_log( "Channel {$channel} failed: {$error}" );
    }
}
```

#### `kdc_qtap_register_notification_channel( $channel_id, $args )`

Register a custom notification channel.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `channel_id` | string | Unique channel identifier |
| `label` | string | Human-readable name |
| `description` | string | Channel description |
| `callback` | callable | Function to send the notification |
| `icon` | string | Dashicon class for admin UI |
| `enabled` | bool | Whether enabled by default |
| `settings` | array | Channel-specific settings |

**Example:**

```php
add_action( 'kdc_qtap_notification_init', function() {
    kdc_qtap_register_notification_channel(
        'twilio_sms',
        array(
            'label'       => __( 'Twilio SMS', 'my-plugin' ),
            'description' => __( 'Send SMS via Twilio', 'my-plugin' ),
            'callback'    => 'my_plugin_send_twilio_sms',
            'icon'        => 'dashicons-smartphone',
            'enabled'     => true,
            'settings'    => array(
                'account_sid' => '',
                'auth_token'  => '',
                'from_number' => '',
            ),
        )
    );
} );
```

#### `kdc_qtap_register_notification_type( $type_id, $args )`

Register a custom notification type.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `type_id` | string | Unique type identifier |
| `label` | string | Human-readable name |
| `description` | string | Type description |
| `default_channels` | array | Default channels for this type |
| `default_priority` | string | Default priority level |

**Example:**

```php
add_action( 'kdc_qtap_notification_init', function() {
    kdc_qtap_register_notification_type(
        'license_expiry',
        array(
            'label'            => __( 'License Expiry', 'my-plugin' ),
            'description'      => __( 'Notifications about expiring licenses', 'my-plugin' ),
            'default_channels' => array( 'email', 'admin_notice' ),
            'default_priority' => 'high',
        )
    );
} );
```

#### `kdc_qtap_notification_channel_available( $channel_id )`

Check if a channel is registered and enabled.

```php
if ( kdc_qtap_notification_channel_available( 'sms' ) ) {
    // SMS channel is available
}
```

#### `kdc_qtap_get_notification_channels( $enabled_only = true )`

Get all registered channels.

```php
$channels = kdc_qtap_get_notification_channels();
// Returns array of channel configurations
```

### WhatsApp Validation Helpers (v2.2.32+)

Use these functions in child plugin forms to validate WhatsApp fields before submission.

#### `kdc_qtap_validate_whatsapp_template( $template_value )`

Validate WhatsApp template field format.

```php
$result = kdc_qtap_validate_whatsapp_template( 'order_status|en' );

// Returns:
array(
    'valid'    => true,           // Whether format is valid
    'template' => 'order_status', // Parsed template name
    'language' => 'en',           // Parsed language code
    'error'    => '',             // Error message if invalid
)

// Example with error:
$result = kdc_qtap_validate_whatsapp_template( 'order-status' );
// 'valid' => false
// 'error' => 'Template must include language code in format: template_name|language_code'
```

#### `kdc_qtap_validate_whatsapp_buttons( $buttons_string )`

Validate WhatsApp button(s) - accepts single button or multiline string.

**Supported button types:**
| Type | Description | Value Required |
|------|-------------|----------------|
| `reply` | Quick reply button | Yes (payload) |
| `url` | URL button | Yes (URL suffix) |
| `phone` | Phone number button | Yes (phone number) |
| `call` | Voice call permission | No |
| `code` | Copy code/coupon button | Yes (code) |
| `flow` | WhatsApp Flow trigger | Yes (flow token) |

```php
// Single button
$result = kdc_qtap_validate_whatsapp_buttons( 'url|https://example.com' );

// Returns:
array(
    'valid'   => true,
    'buttons' => array(
        array( 'type' => 'url', 'value' => 'https://example.com' ),
    ),
    'errors'  => array(),
)

// Multiple buttons (multiline)
$buttons = "url|https://example.com\nreply|confirm\ncall";
$result = kdc_qtap_validate_whatsapp_buttons( $buttons );

// Returns:
array(
    'valid'   => true,  // Whether ALL buttons are valid
    'buttons' => array(
        array( 'type' => 'url', 'value' => 'https://example.com' ),
        array( 'type' => 'reply', 'value' => 'confirm' ),
        array( 'type' => 'call', 'value' => '' ),
    ),
    'errors'  => array(), // Errors keyed by line number (1-indexed)
)

// With errors:
$buttons = "url|https://example.com\ninvalid_type|value";
$result = kdc_qtap_validate_whatsapp_buttons( $buttons );
// 'valid' => false
// 'errors' => array( 2 => 'Invalid button type. Valid types: reply, url, phone, call, code, flow, order' )
```

#### `kdc_qtap_get_whatsapp_button_types()`

Get button type metadata for building form dropdowns.

```php
$types = kdc_qtap_get_whatsapp_button_types();

// Returns:
array(
    'reply' => array(
        'label'          => 'Quick Reply',
        'description'    => 'Quick reply button with payload',
        'value_required' => true,
        'value_label'    => 'Payload',
    ),
    'url' => array(
        'label'          => 'URL',
        'description'    => 'URL button (dynamic suffix)',
        'value_required' => true,
        'value_label'    => 'URL',
    ),
    'phone' => array(
        'label'          => 'Phone Number',
        'description'    => 'Phone number button (dynamic)',
        'value_required' => true,
        'value_label'    => 'Phone Number',
    ),
    'call' => array(
        'label'          => 'Call Permission',
        'description'    => 'Voice call permission request',
        'value_required' => false,
        'value_label'    => '',
    ),
    'code' => array(
        'label'          => 'Copy Code',
        'description'    => 'Coupon/discount code button',
        'value_required' => true,
        'value_label'    => 'Code',
    ),
    'flow' => array(
        'label'          => 'WhatsApp Flow',
        'description'    => 'Trigger a WhatsApp Flow',
        'value_required' => true,
        'value_label'    => 'Flow Token',
    ),
)
```

#### Example: Child Plugin Form Validation

```php
// In your child plugin's form handler:
add_action( 'wp_ajax_my_plugin_save_notification', function() {
    $template = sanitize_text_field( $_POST['whatsapp_template'] );
    $buttons  = sanitize_textarea_field( $_POST['whatsapp_buttons'] );
    
    // Validate template
    $template_result = kdc_qtap_validate_whatsapp_template( $template );
    if ( ! $template_result['valid'] ) {
        wp_send_json_error( array(
            'field'   => 'whatsapp_template',
            'message' => $template_result['error'],
        ) );
    }
    
    // Validate buttons
    $buttons_result = kdc_qtap_validate_whatsapp_buttons( $buttons );
    if ( ! $buttons_result['valid'] ) {
        wp_send_json_error( array(
            'field'   => 'whatsapp_buttons',
            'errors'  => $buttons_result['errors'],
        ) );
    }
    
    // All valid - save the data
    update_option( 'my_plugin_whatsapp_template', $template );
    update_option( 'my_plugin_whatsapp_buttons', $buttons );
    
    wp_send_json_success();
} );
```

### WhatsApp Form Field Helpers (v2.3.3+)

Use these functions to render consistent WhatsApp form fields across all child plugins. Labels, placeholders, and descriptions are provided by the parent plugin.

#### `kdc_qtap_render_whatsapp_template_field( $value, $args )`

Render a template name/language input field.

```php
<?php kdc_qtap_render_whatsapp_template_field( $saved_template ); ?>

// With options:
<?php kdc_qtap_render_whatsapp_template_field( $saved_template, array(
    'name'     => 'notification_template',  // Custom name attribute
    'id'       => 'my-template',            // Custom id (defaults to name)
    'required' => true,                     // Add required attribute
    'echo'     => false,                    // Return HTML instead of echo
) ); ?>
```

#### `kdc_qtap_render_whatsapp_body_field( $value, $args )`

Render a body variables textarea.

```php
<?php kdc_qtap_render_whatsapp_body_field( $saved_body ); ?>

// With options:
<?php kdc_qtap_render_whatsapp_body_field( $saved_body, array(
    'name'     => 'notification_body',
    'rows'     => 6,                        // Textarea rows (default: 4)
    'required' => true,
) ); ?>
```

#### `kdc_qtap_render_whatsapp_header_field( $value, $args )`

Render a header variable input field.

```php
<?php kdc_qtap_render_whatsapp_header_field( $saved_header ); ?>

// With options:
<?php kdc_qtap_render_whatsapp_header_field( $saved_header, array(
    'name'     => 'notification_header',
    'required' => false,
) ); ?>
```

**Header Variable Formats (v2.3.20+):**

The header variable automatically detects the type based on the value:

| Format | Type | Example | Payload |
|--------|------|---------|---------|
| Plain text | `text` | `Order Confirmation` | `{ "type": "text", "text": "Order Confirmation" }` |
| Coordinates | `location` | `19.0760,72.8777` | `{ "type": "location", "location": { "latitude": 19.0760, "longitude": 72.8777 } }` |
| Coordinates + info | `location` | `19.0760,72.8777\|School Name\|123 Main St` | `{ "type": "location", "location": { "latitude": 19.0760, "longitude": 72.8777, "name": "School Name", "address": "123 Main St" } }` |
| Image URL | `image` | `https://example.com/logo.png` | `{ "type": "image", "image": { "link": "..." } }` |
| Video URL | `video` | `https://example.com/intro.mp4` | `{ "type": "video", "video": { "link": "..." } }` |
| Other URL | `document` | `https://example.com/fee.pdf` | `{ "type": "document", "document": { "link": "..." } }` |

**Location Format:**
```
lat,long
lat,long|name
lat,long|name|address
```

**Supported Media Extensions:**
- **Image**: jpg, jpeg, png, gif, webp, bmp
- **Video**: mp4, avi, mov, mkv, webm, 3gp, m4v
- **Document**: pdf, doc, docx, xls, xlsx, ppt, pptx, and any other URL

**Examples:**
```php
// Text header
'whatsapp_header' => 'Order #12345 Confirmed',

// Location header
'whatsapp_header' => '19.0760,72.8777|ABC School|123 Education Lane, Mumbai',

// Image header
'whatsapp_header' => 'https://example.com/school-logo.png',

// Video header  
'whatsapp_header' => 'https://example.com/welcome-video.mp4',

// Document header
'whatsapp_header' => 'https://example.com/fee-receipt.pdf',
```

#### `kdc_qtap_render_whatsapp_footer_field( $value, $args )`

Render a footer variable input field.

```php
<?php kdc_qtap_render_whatsapp_footer_field( $saved_footer ); ?>

// With options:
<?php kdc_qtap_render_whatsapp_footer_field( $saved_footer, array(
    'name'     => 'notification_footer',
    'required' => false,
) ); ?>
```

#### `kdc_qtap_render_whatsapp_buttons_field( $value, $args )`

Render a button variables textarea with supported types in description.

```php
<?php kdc_qtap_render_whatsapp_buttons_field( $saved_buttons ); ?>

// With options:
<?php kdc_qtap_render_whatsapp_buttons_field( $saved_buttons, array(
    'name'     => 'notification_buttons',
    'rows'     => 5,
    'required' => false,
) ); ?>
```

#### Complete Form Example

```php
<form method="post" action="">
    <?php wp_nonce_field( 'my_plugin_notification', 'my_plugin_nonce' ); ?>
    
    <table class="form-table">
        <?php kdc_qtap_render_whatsapp_template_field( 
            get_option( 'my_plugin_template', '' ),
            array( 'name' => 'my_plugin_template', 'required' => true )
        ); ?>
        
        <?php kdc_qtap_render_whatsapp_header_field( 
            get_option( 'my_plugin_header', '' ),
            array( 'name' => 'my_plugin_header' )
        ); ?>
        
        <?php kdc_qtap_render_whatsapp_body_field( 
            get_option( 'my_plugin_body', '' ),
            array( 'name' => 'my_plugin_body' )
        ); ?>
        
        <?php kdc_qtap_render_whatsapp_footer_field( 
            get_option( 'my_plugin_footer', '' ),
            array( 'name' => 'my_plugin_footer' )
        ); ?>
        
        <?php kdc_qtap_render_whatsapp_buttons_field( 
            get_option( 'my_plugin_buttons', '' ),
            array( 'name' => 'my_plugin_buttons' )
        ); ?>
    </table>
    
    <p class="submit">
        <button type="submit" class="button button-primary">
            <?php esc_html_e( 'Save Settings', 'my-plugin' ); ?>
        </button>
    </p>
</form>
```

## Built-in Channels

### Email Channel (`email`)

Sends notifications via WordPress `wp_mail()` with HTML formatting.

**Recipient extraction:**
- String: Used directly as email
- Array with `email` key: Uses that value
- Array with `user_id` key: Gets email from WordPress user

### Admin Notice Channel (`admin_notice`)

Displays notifications as WordPress admin notices.

**Behavior:**
- Stored in transient for display
- Priority maps to notice type (high/urgent = warning, low = info)
- Alert type = error notice

### Webhook Channel (`webhook`)

Sends notifications to external URLs (Slack, Discord, Zapier, etc.).

**Payload format:**
```json
{
    "subject": "Notification Subject",
    "message": "Notification message",
    "type": "alert",
    "priority": "high",
    "timestamp": "2025-01-03 12:00:00",
    "source": "my-plugin",
    "data": {}
}
```

### Log Channel (`log`)

Logs notifications without sending (for debugging).

**Behavior:**
- Writes to WordPress debug log when `WP_DEBUG` is enabled
- Always returns success

## Available Hooks

### Actions

#### `kdc_qtap_notification_init`

Fires when the notification system initializes. **Use this to register channels and types.**

```php
add_action( 'kdc_qtap_notification_init', function( $notifications ) {
    // Register custom channels
    $notifications->register_channel( 'my_channel', array( ... ) );
    
    // Register custom types
    $notifications->register_type( 'my_type', array( ... ) );
} );
```

#### `kdc_qtap_notification_before_send`

Fires before a notification is sent.

```php
add_action( 'kdc_qtap_notification_before_send', function( $notification ) {
    // Log or modify before sending
    error_log( 'Sending: ' . $notification['subject'] );
} );
```

#### `kdc_qtap_notification_sent`

Fires after a notification is successfully sent via a channel.

```php
add_action( 'kdc_qtap_notification_sent', function( $channel_id, $notification, $result ) {
    // Track successful sends
}, 10, 3 );
```

#### `kdc_qtap_notification_failed`

Fires when a notification fails to send.

```php
add_action( 'kdc_qtap_notification_failed', function( $channel_id, $notification, $error ) {
    // Handle failures, retry logic, etc.
    error_log( "Notification failed on {$channel_id}: {$error}" );
}, 10, 3 );
```

#### `kdc_qtap_notification_log`

Fires when a notification is logged. Use for custom logging.

```php
add_action( 'kdc_qtap_notification_log', function( $log_entry, $notification, $results ) {
    // Store in database, send to analytics, etc.
    global $wpdb;
    $wpdb->insert( $wpdb->prefix . 'my_notification_log', array(
        'subject'   => $notification['subject'],
        'channels'  => wp_json_encode( $results['channels'] ),
        'timestamp' => current_time( 'mysql' ),
    ) );
}, 10, 3 );
```

### Filters

#### `kdc_qtap_notification_channels`

Modify registered channels.

```php
add_filter( 'kdc_qtap_notification_channels', function( $channels ) {
    // Remove a channel
    unset( $channels['admin_notice'] );
    return $channels;
} );
```

#### `kdc_qtap_notification_types`

Modify registered notification types.

```php
add_filter( 'kdc_qtap_notification_types', function( $types ) {
    // Modify default channels for a type
    $types['system']['default_channels'] = array( 'email', 'slack' );
    return $types;
} );
```

#### `kdc_qtap_notification_data`

Modify notification data before sending.

```php
add_filter( 'kdc_qtap_notification_data', function( $notification ) {
    // Add tracking ID to all notifications
    $notification['data']['tracking_id'] = uniqid( 'notif_' );
    return $notification;
} );
```

#### `kdc_qtap_notification_should_send`

Control whether a notification should be sent.

```php
add_filter( 'kdc_qtap_notification_should_send', function( $should_send, $notification ) {
    // Block notifications during maintenance
    if ( get_option( 'maintenance_mode' ) ) {
        return false;
    }
    
    // Block low priority at night
    if ( $notification['priority'] === 'low' && date( 'H' ) >= 22 ) {
        return false;
    }
    
    return $should_send;
}, 10, 2 );
```

#### `kdc_qtap_notification_message`

Modify message before sending to a specific channel.

```php
add_filter( 'kdc_qtap_notification_message', function( $message, $channel_id, $notification ) {
    // Add channel-specific formatting
    if ( $channel_id === 'slack' ) {
        $message = "*{$notification['subject']}*\n\n{$message}";
    }
    
    return $message;
}, 10, 3 );
```

#### `kdc_qtap_notification_recipient`

Modify recipient before sending to a specific channel.

```php
add_filter( 'kdc_qtap_notification_recipient', function( $recipient, $channel_id, $notification ) {
    // Override recipient for testing
    if ( defined( 'WP_DEBUG' ) && WP_DEBUG && $channel_id === 'email' ) {
        return 'debug@example.com';
    }
    
    return $recipient;
}, 10, 3 );
```

#### `kdc_qtap_notification_template`

Modify message templates.

```php
add_filter( 'kdc_qtap_notification_template', function( $message, $template_id, $data ) {
    if ( $template_id === 'order_shipped' ) {
        return "Great news! Your order #{$data['order_id']} is on its way!\n\n" .
               "Track your package: {$data['tracking_url']}\n\n" .
               "Thank you for shopping with us!";
    }
    
    return $message;
}, 10, 3 );
```

#### `kdc_qtap_notification_webhook_payload`

Modify webhook payload before sending.

```php
add_filter( 'kdc_qtap_notification_webhook_payload', function( $payload, $notification ) {
    // Format for Slack
    return array(
        'text' => $notification['subject'],
        'blocks' => array(
            array(
                'type' => 'section',
                'text' => array(
                    'type' => 'mrkdwn',
                    'text' => $notification['message'],
                ),
            ),
        ),
    );
}, 10, 2 );
```

#### `kdc_qtap_notification_log_enabled`

Control logging per notification.

```php
add_filter( 'kdc_qtap_notification_log_enabled', function( $enabled, $notification ) {
    // Don't log info notifications
    if ( $notification['type'] === 'info' ) {
        return false;
    }
    
    return $enabled;
}, 10, 2 );
```

## Creating Custom Channels

### Example: Twilio SMS Channel

```php
<?php
/**
 * Register Twilio SMS channel
 */
add_action( 'kdc_qtap_notification_init', function() {
    kdc_qtap_register_notification_channel(
        'twilio_sms',
        array(
            'label'       => __( 'SMS (Twilio)', 'my-plugin' ),
            'description' => __( 'Send SMS notifications via Twilio', 'my-plugin' ),
            'callback'    => 'my_plugin_send_twilio_sms',
            'icon'        => 'dashicons-smartphone',
            'enabled'     => true,
            'settings'    => array(
                'account_sid' => get_option( 'my_plugin_twilio_sid' ),
                'auth_token'  => get_option( 'my_plugin_twilio_token' ),
                'from_number' => get_option( 'my_plugin_twilio_number' ),
            ),
        )
    );
} );

/**
 * Twilio SMS callback
 *
 * @param array $notification Notification data.
 * @param array $channel      Channel configuration.
 * @return bool Success or failure.
 */
function my_plugin_send_twilio_sms( $notification, $channel ) {
    // Extract phone number from recipient
    $phone = '';
    if ( is_array( $notification['recipient'] ) ) {
        $phone = isset( $notification['recipient']['phone'] ) 
            ? $notification['recipient']['phone'] 
            : '';
    } else {
        // Assume it's a phone number if it starts with +
        if ( strpos( $notification['recipient'], '+' ) === 0 ) {
            $phone = $notification['recipient'];
        }
    }
    
    if ( empty( $phone ) ) {
        return false;
    }
    
    // Get Twilio credentials
    $sid   = $channel['settings']['account_sid'];
    $token = $channel['settings']['auth_token'];
    $from  = $channel['settings']['from_number'];
    
    if ( empty( $sid ) || empty( $token ) || empty( $from ) ) {
        return false;
    }
    
    // Build message (SMS is text-only, no subject)
    $body = $notification['message'];
    if ( ! empty( $notification['subject'] ) ) {
        $body = $notification['subject'] . ': ' . $body;
    }
    
    // Twilio API request
    $response = wp_remote_post(
        "https://api.twilio.com/2010-04-01/Accounts/{$sid}/Messages.json",
        array(
            'headers' => array(
                'Authorization' => 'Basic ' . base64_encode( "{$sid}:{$token}" ),
            ),
            'body' => array(
                'To'   => $phone,
                'From' => $from,
                'Body' => $body,
            ),
        )
    );
    
    if ( is_wp_error( $response ) ) {
        return false;
    }
    
    $code = wp_remote_retrieve_response_code( $response );
    return $code >= 200 && $code < 300;
}
```

### Example: Slack Channel

```php
<?php
/**
 * Register Slack channel
 */
add_action( 'kdc_qtap_notification_init', function() {
    kdc_qtap_register_notification_channel(
        'slack',
        array(
            'label'       => __( 'Slack', 'my-plugin' ),
            'description' => __( 'Send notifications to Slack', 'my-plugin' ),
            'callback'    => 'my_plugin_send_slack',
            'icon'        => 'dashicons-format-chat',
            'enabled'     => true,
            'settings'    => array(
                'webhook_url' => get_option( 'my_plugin_slack_webhook' ),
                'channel'     => get_option( 'my_plugin_slack_channel', '#notifications' ),
            ),
        )
    );
} );

/**
 * Slack notification callback
 */
function my_plugin_send_slack( $notification, $channel ) {
    $webhook_url = $channel['settings']['webhook_url'];
    
    if ( empty( $webhook_url ) ) {
        return false;
    }
    
    // Build Slack payload
    $payload = array(
        'channel'  => $channel['settings']['channel'],
        'username' => get_bloginfo( 'name' ),
        'text'     => $notification['subject'],
        'attachments' => array(
            array(
                'color'  => my_plugin_get_slack_color( $notification['priority'] ),
                'text'   => $notification['message'],
                'footer' => 'via qTap Notifications',
                'ts'     => time(),
            ),
        ),
    );
    
    $response = wp_remote_post( $webhook_url, array(
        'headers' => array( 'Content-Type' => 'application/json' ),
        'body'    => wp_json_encode( $payload ),
    ) );
    
    return ! is_wp_error( $response ) && 
           wp_remote_retrieve_response_code( $response ) === 200;
}

function my_plugin_get_slack_color( $priority ) {
    switch ( $priority ) {
        case 'urgent': return '#d63638';
        case 'high':   return '#dba617';
        case 'low':    return '#72aee6';
        default:       return '#00a32a';
    }
}
```

## Template System

The notification system supports `{{placeholder}}` syntax for dynamic content.

### Centralized Variable Replacement (v2.3.11+)

Variables are now processed **centrally and channel-agnostically** before sending to ANY channel. This means `{{variables}}` work consistently across:

- **Email**: `subject`, `message`
- **WhatsApp**: `whatsapp_template`, `whatsapp_header`, `whatsapp_body`, `whatsapp_footer`, `whatsapp_buttons`
- **SMS**: `message`
- **Webhook**: `message`

**Example - Same variables work everywhere:**

```php
kdc_qtap_send_notification( array(
    'recipient' => array(
        'email' => 'user@example.com',
        'phone' => '+919876543210',
    ),
    // Email fields - variables work!
    'subject' => 'Payment Due: {{amount}}',
    'message' => 'Hello {{customer_name}}, please pay {{amount}} by {{due_date}}.',
    // WhatsApp fields - same variables work!
    'whatsapp_template' => 'payment_reminder|en',
    'whatsapp_body'     => "{{customer_name}}\n{{amount}}\n{{due_date}}",
    'whatsapp_buttons'  => "url|{{payment_url}}",
    // Variable data
    'data' => array(
        'customer_name' => 'John Doe',
        'amount'        => '₹5,000',
        'due_date'      => '15 Jan 2026',
        'payment_url'   => 'https://pay.example.com/inv/123',
    ),
    'channels' => array( 'email', 'whatsapp' ),
) );
```

### Customize Processed Fields

```php
// Add custom fields to variable processing
add_filter( 'kdc_qtap_notification_variable_fields', function( $fields, $notification ) {
    $fields[] = 'custom_field';
    return $fields;
}, 10, 2 );
```

### Built-in Placeholders

All built-in placeholders support format modifiers (e.g., `{{site_name:upper}}`).

| Placeholder | Description |
|-------------|-------------|
| `{{site_name}}` | WordPress site name |
| `{{site_url}}` | Site home URL |
| `{{site_icon}}` | Site favicon/icon URL |
| `{{site_logo}}` | Site logo URL |
| `{{admin_email}}` | Admin email address |
| `{{current_year}}` | Current year (e.g., 2026) |

### Custom Data Placeholders

Any key in the `data` array becomes a placeholder:

```php
kdc_qtap_send_notification( array(
    'recipient' => 'user@example.com',
    'subject'   => 'Order #{{order_id}} Shipped',
    'message'   => 'Hello {{customer_name}}, your order #{{order_id}} has shipped via {{carrier}}. Track it at {{tracking_url}}',
    'data'      => array(
        'order_id'      => 12345,
        'customer_name' => 'John Doe',
        'carrier'       => 'FedEx',
        'tracking_url'  => 'https://fedex.com/track/ABC123',
    ),
) );
```

### Custom Templates via Filter

```php
add_filter( 'kdc_qtap_notification_template', function( $message, $template_id, $data ) {
    $templates = array(
        'welcome' => "Welcome to {{site_name}}, {{name}}!\n\nYour account has been created.",
        'order_shipped' => "Your order #{{order_id}} has shipped!\n\nTracking: {{tracking_url}}",
        'password_reset' => "Click here to reset your password: {{reset_url}}",
    );
    
    if ( isset( $templates[ $template_id ] ) ) {
        return $templates[ $template_id ];
    }
    
    return $message;
}, 10, 3 );

// Usage
kdc_qtap_send_notification( array(
    'recipient' => 'user@example.com',
    'subject'   => 'Welcome!',
    'message'   => '', // Will be replaced by template
    'template'  => 'welcome',
    'data'      => array( 'name' => 'John' ),
) );
```

## Best Practices

### 1. Always Specify Source

```php
kdc_qtap_send_notification( array(
    // ...
    'source' => 'my-child-plugin', // Helps with debugging and logging
) );
```

### 2. Use Notification Types

```php
// Register once
add_action( 'kdc_qtap_notification_init', function() {
    kdc_qtap_register_notification_type( 'my_plugin_alert', array(
        'label'            => 'My Plugin Alerts',
        'default_channels' => array( 'email', 'admin_notice' ),
        'default_priority' => 'high',
    ) );
} );

// Use everywhere
kdc_qtap_send_notification( array(
    'recipient' => $email,
    'subject'   => 'Alert!',
    'message'   => $message,
    'type'      => 'my_plugin_alert', // Uses registered defaults
) );
```

### 3. Check Channel Availability

```php
// Before depending on a channel
if ( kdc_qtap_notification_channel_available( 'sms' ) ) {
    $channels[] = 'sms';
}
```

### 4. Handle Failures Gracefully

```php
$result = kdc_qtap_send_notification( $args );

if ( ! $result['success'] ) {
    // Log errors
    foreach ( $result['errors'] as $channel => $error ) {
        error_log( "Notification failed on {$channel}: {$error}" );
    }
    
    // Maybe retry or fallback
    if ( ! $result['channels']['email'] ) {
        // Fallback to admin notice
        kdc_qtap_send_notification( array(
            'recipient' => get_current_user_id(),
            'subject'   => $args['subject'],
            'message'   => $args['message'],
            'channels'  => array( 'admin_notice' ),
        ) );
    }
}
```

### 5. Use Data for Dynamic Content

```php
// Instead of string concatenation
kdc_qtap_send_notification( array(
    'message' => 'Order #' . $order_id . ' shipped', // ❌ Hard to modify
) );

// Use data and placeholders
kdc_qtap_send_notification( array(
    'message' => 'Order #{{order_id}} shipped', // ✅ Filterable
    'data'    => array( 'order_id' => $order_id ),
) );
```

## Troubleshooting

### Notifications Not Sending

1. Check if channel is enabled: `kdc_qtap_notification_channel_available( 'email' )`
2. Check for filter blocking: Look for `kdc_qtap_notification_should_send` filters
3. Enable WP_DEBUG and check error logs
4. Use the `log` channel to verify notification is being processed
5. **v2.0.0+**: Check the Notification Log tab in qTap Dashboard for status and errors

### Email Not Arriving

1. Test WordPress email: `wp_mail( 'test@example.com', 'Test', 'Test' )`
2. Check spam folder
3. Consider using an SMTP plugin
4. Check `wp_mail_failed` action for errors

### Webhook Failing

1. Test URL manually with curl/Postman
2. Check response code via the `kdc_qtap_notification_failed` action
3. Verify payload format matches endpoint expectations
4. Check firewall/security rules

## Notification Log (v2.0.0+)

Starting with v2.0.0, all notifications are automatically logged to a custom database table for persistent storage and review.

### Accessing the Log

Navigate to **qTap App → Notification Log** in the WordPress admin.

### Log Features

- **Statistics Dashboard**: View total, sent, failed, and pending counts at a glance
- **Advanced Filtering**: Filter by status, type, priority, date range, and search
- **Sortable Columns**: Sort by date, type, priority, or status
- **Detail Modal**: Click "View" to see full notification details including message, recipient, channel results
- **Bulk Actions**: Delete multiple entries or resend failed notifications
- **Export**: Download log as CSV for external analysis
- **Settings**: Configure log retention period (1-365 days)

### Programmatic Access

```php
// Get the notification log instance
$log = kdc_qtap_notification_log();

// Query logs with filters
$result = $log->query( array(
    'page'      => 1,
    'per_page'  => 20,
    'status'    => 'failed',  // 'sent', 'failed', 'pending'
    'type'      => 'order_shipped',
    'priority'  => 'high',
    'search'    => 'customer@example.com',
    'date_from' => '2026-01-01',
    'date_to'   => '2026-01-31',
    'orderby'   => 'created_at',
    'order'     => 'DESC',
) );

// $result contains:
// - items: Array of log entries
// - total: Total count
// - pages: Total pages
// - page: Current page
// - per_page: Items per page

// Get a single log entry by ID
$entry = $log->get( 123 );

// Get statistics
$stats = $log->get_stats();
// Returns: total, sent, failed, pending, by_type

// Get distinct notification types
$types = $log->get_types();

// Manual cleanup (normally runs via daily cron)
$deleted = $log->cleanup( 30 ); // Delete entries older than 30 days

// Clear all logs
$log->clear_all();
```

### Log Entry Structure

Each log entry contains:

| Field | Type | Description |
|-------|------|-------------|
| `id` | int | Auto-increment ID |
| `notification_id` | string | UUID for the notification |
| `type` | string | Notification type |
| `subject` | string | Subject/title |
| `message` | text | Message body |
| `recipient` | text | Recipient(s) (JSON if array) |
| `channels` | text | Channels used (JSON array) |
| `data` | text | Additional data (JSON) |
| `priority` | string | low/normal/high/urgent |
| `status` | string | sent/failed/pending |
| `results` | text | Per-channel results (JSON) |
| `error_message` | text | Error message if failed |
| `retry_count` | int | Number of retry attempts |
| `created_at` | datetime | When notification was sent |
| `updated_at` | datetime | Last update time |

## Changelog

### 2.0.0
- **Major**: Notification Log System with database storage
- Added custom database table `{prefix}_kdc_qtap_notification_log`
- Added "Notification Log" admin tab with full log viewer
- Added statistics cards, filters, sorting, pagination
- Added bulk actions (delete, resend failed)
- Added single actions (view details modal, resend, delete)
- Added export to CSV functionality
- Added log settings (enable/disable, retention period)
- Added daily cron job for automatic log cleanup
- All notifications automatically logged to database

### 1.9.21
- Initial release of notification system
- Built-in channels: email, admin_notice, webhook, log
- 15+ hooks for customization
- Template system with placeholder support
- Multi-recipient and multi-channel support
