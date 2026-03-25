# Notification Template Variables - Child Plugin Guide

**Version:** 2.0.5  
**Parent Plugin:** qTap App

## Overview

The qTap Notification Template Variables system allows you to create dynamic notification content using `{{variable}}` placeholders. Variables are automatically resolved when notifications are sent.

## Quick Start

### Using Built-in Variables

```php
// Send notification with template variables
kdc_qtap_send_notification( array(
    'type'      => 'welcome',
    'recipient' => array( 'email' => 'user@example.com', 'user_id' => 123 ),
    'subject'   => 'Welcome to {{site_name}}!',
    'message'   => 'Hello {{user_name}},

Welcome to {{site_name}}! Your account was created on {{date}}.

Your username: {{user_login}}
Your email: {{user_email}}

Visit us at {{site_url}}

Best regards,
{{site_name}} Team',
) );
```

### Processing Templates Manually

```php
$template = 'Hello {{user_name}}, your order #{{order_id}} has shipped!';
$context  = array(
    'user_id'  => 123,
    'order_id' => 'WC-5678',
);

$message = kdc_qtap_process_notification_template( $template, $context );
// Output: "Hello John Doe, your order #WC-5678 has shipped!"
```

---

## Built-in Variables

### Site Variables (`site` group)

| Variable | Description | Example |
|----------|-------------|---------|
| `{{site_name}}` | Website name | My Store |
| `{{site_url}}` | Website URL | https://example.com |
| `{{admin_email}}` | Admin email address | admin@example.com |
| `{{site_description}}` | Website tagline | Your tagline here |

### User Variables (`user` group)

| Variable | Description |
|----------|-------------|
| `{{user_id}}` | User ID (from context or current user) |
| `{{user_email}}` | User email address |
| `{{user_name}}` | User display name |
| `{{user_login}}` | Username |
| `{{user_first_name}}` | First name |
| `{{user_last_name}}` | Last name |
| `{{user_role}}` | Primary user role |

### Post Variables (`post` group)

Requires `post_id` in context.

| Variable | Description |
|----------|-------------|
| `{{post_id}}` | Post ID |
| `{{post_title}}` | Post title |
| `{{post_url}}` | Post permalink |
| `{{post_excerpt}}` | Post excerpt |
| `{{post_type}}` | Post type (post, page, etc.) |
| `{{post_status}}` | Post status |
| `{{post_author_name}}` | Author display name |

### Date/Time Variables (`date` group)

| Variable | Description | Example |
|----------|-------------|---------|
| `{{date}}` | Current date (site format) | January 3, 2026 |
| `{{time}}` | Current time (site format) | 5:30 am |
| `{{datetime}}` | Date and time combined | January 3, 2026 5:30 am |
| `{{year}}` | Current year | 2026 |
| `{{month}}` | Current month name | January |
| `{{day}}` | Day of month | 3 |

### Notification Variables (`notification` group)

| Variable | Description |
|----------|-------------|
| `{{notification_type}}` | Notification type identifier |
| `{{notification_priority}}` | Priority level (low, normal, high, urgent) |

---

## WordPress Meta Variables (`wp_meta_*`)

The `wp_meta_` prefix enables dynamic access to any WordPress data.

### User Data (`wp_meta_user_*`)

Access user fields or meta by name:

```
{{wp_meta_user_email}}        → User email
{{wp_meta_user_display_name}} → Display name
{{wp_meta_user_first_name}}   → First name
{{wp_meta_user_nickname}}     → Nickname
{{wp_meta_user_billing_phone}} → WooCommerce billing phone (if exists)
```

### User Meta (`wp_meta_usermeta_*`)

Access any user meta key:

```
{{wp_meta_usermeta_billing_address_1}} → WooCommerce billing address
{{wp_meta_usermeta_custom_field}}      → Any custom user meta
{{wp_meta_usermeta__wc_memberships_membership_id}} → WooCommerce membership
```

### Post Data (`wp_meta_post_*`)

Access post fields:

```
{{wp_meta_post_title}}       → Post title
{{wp_meta_post_url}}         → Permalink
{{wp_meta_post_permalink}}   → Permalink (alias)
{{wp_meta_post_content}}     → Post content
{{wp_meta_post_excerpt}}     → Excerpt
{{wp_meta_post_status}}      → Status
{{wp_meta_post_type}}        → Post type
{{wp_meta_post_author_name}} → Author display name
{{wp_meta_post_author_email}}→ Author email
{{wp_meta_post_featured_image}} → Featured image URL
```

### Post Meta (`wp_meta_postmeta_*`)

Access any post meta key:

```
{{wp_meta_postmeta__price}}           → WooCommerce product price
{{wp_meta_postmeta__sku}}             → Product SKU
{{wp_meta_postmeta_custom_field}}     → Any custom post meta
```

### WordPress Options (`wp_meta_option_*`)

Access any option from `wp_options`:

```
{{wp_meta_option_blogname}}        → Site name
{{wp_meta_option_admin_email}}     → Admin email
{{wp_meta_option_date_format}}     → Date format setting
{{wp_meta_option_timezone_string}} → Timezone
{{wp_meta_option_woocommerce_currency}} → WooCommerce currency
```

### Site Info (`wp_meta_site_*`)

Access `get_bloginfo()` values:

```
{{wp_meta_site_name}}        → Site name
{{wp_meta_site_description}} → Tagline
{{wp_meta_site_url}}         → Site URL
{{wp_meta_site_wpurl}}       → WordPress URL
{{wp_meta_site_charset}}     → Character set
{{wp_meta_site_version}}     → WordPress version
{{wp_meta_site_language}}    → Site language
```

### Date Formatting (`wp_meta_date_*`)

Format dates with named or custom formats:

```
{{wp_meta_date_full}}      → January 3, 2026 5:30 am
{{wp_meta_date_short}}     → 01/03/2026
{{wp_meta_date_iso}}       → ISO 8601 format
{{wp_meta_date_mysql}}     → 2026-01-03 05:30:00
{{wp_meta_date_timestamp}} → Unix timestamp
{{wp_meta_date_rss}}       → RSS format
{{wp_meta_date_Y-m-d}}     → 2026-01-03 (custom format)
{{wp_meta_date_F j, Y}}    → January 3, 2026 (custom format)
```

---

## Registering Custom Variables

### Register a Single Variable

```php
add_action( 'kdc_qtap_register_notification_variables', function( $variables ) {
    $variables->register_variable( 'license_key', array(
        'label'       => __( 'License Key', 'my-plugin' ),
        'description' => __( 'The customer license key', 'my-plugin' ),
        'group'       => 'license',  // Custom group
        'example'     => 'XXXX-XXXX-XXXX-XXXX',
        'callback'    => function( $context ) {
            // Resolve the variable value based on context
            if ( isset( $context['license_key'] ) ) {
                return $context['license_key'];
            }
            
            // Or fetch from database
            if ( isset( $context['license_id'] ) ) {
                return get_post_meta( $context['license_id'], '_license_key', true );
            }
            
            return '';
        },
    ) );
} );
```

### Register a Variable Group

```php
add_action( 'kdc_qtap_register_notification_variables', function( $variables ) {
    $variables->register_group( 'license', array(
        'label'       => __( 'License Variables', 'my-plugin' ),
        'description' => __( 'Variables related to software licenses', 'my-plugin' ),
        'icon'        => 'dashicons-admin-network',
        'priority'    => 60, // Ordering (lower = earlier)
    ) );
} );
```

### Complete Example: License Plugin Variables

```php
<?php
/**
 * Register license notification variables.
 */
add_action( 'kdc_qtap_register_notification_variables', function( $variables ) {
    
    // Register the group first
    $variables->register_group( 'license', array(
        'label'       => __( 'License', 'my-license-plugin' ),
        'description' => __( 'License management variables', 'my-license-plugin' ),
        'icon'        => 'dashicons-admin-network',
        'priority'    => 25,
    ) );
    
    // License Key
    $variables->register_variable( 'license_key', array(
        'label'       => __( 'License Key', 'my-license-plugin' ),
        'description' => __( 'The full license key', 'my-license-plugin' ),
        'group'       => 'license',
        'callback'    => function( $context ) {
            return $context['license_key'] ?? '';
        },
    ) );
    
    // License Status
    $variables->register_variable( 'license_status', array(
        'label'       => __( 'License Status', 'my-license-plugin' ),
        'description' => __( 'Current license status (active, expired, etc.)', 'my-license-plugin' ),
        'group'       => 'license',
        'callback'    => function( $context ) {
            if ( isset( $context['license_id'] ) ) {
                return get_post_meta( $context['license_id'], '_status', true );
            }
            return $context['license_status'] ?? '';
        },
    ) );
    
    // License Expiry
    $variables->register_variable( 'license_expiry', array(
        'label'       => __( 'License Expiry Date', 'my-license-plugin' ),
        'description' => __( 'When the license expires', 'my-license-plugin' ),
        'group'       => 'license',
        'callback'    => function( $context ) {
            $expiry = $context['license_expiry'] ?? '';
            if ( $expiry ) {
                return wp_date( get_option( 'date_format' ), strtotime( $expiry ) );
            }
            return __( 'Never', 'my-license-plugin' );
        },
    ) );
    
    // Activations Count
    $variables->register_variable( 'license_activations', array(
        'label'       => __( 'Active Domains', 'my-license-plugin' ),
        'description' => __( 'Number of currently activated domains', 'my-license-plugin' ),
        'group'       => 'license',
        'callback'    => function( $context ) {
            if ( isset( $context['license_id'] ) ) {
                global $wpdb;
                return $wpdb->get_var( $wpdb->prepare(
                    "SELECT COUNT(*) FROM {$wpdb->prefix}license_activations WHERE license_id = %d AND status = 'active'",
                    $context['license_id']
                ) );
            }
            return $context['license_activations'] ?? '0';
        },
    ) );
    
    // Product Name
    $variables->register_variable( 'license_product', array(
        'label'       => __( 'Product Name', 'my-license-plugin' ),
        'description' => __( 'The licensed product name', 'my-license-plugin' ),
        'group'       => 'license',
        'callback'    => function( $context ) {
            if ( isset( $context['product_id'] ) ) {
                return get_the_title( $context['product_id'] );
            }
            return $context['product_name'] ?? '';
        },
    ) );
} );
```

### Using Custom Variables in Notifications

```php
// Send license activation notification
kdc_qtap_send_notification( array(
    'type'      => 'license_activated',
    'recipient' => array( 'user_id' => $user_id ),
    'subject'   => '{{license_product}} License Activated',
    'message'   => 'Hello {{user_name}},

Your license for {{license_product}} has been activated!

License Key: {{license_key}}
Status: {{license_status}}
Expires: {{license_expiry}}
Active Domains: {{license_activations}}

If you have any questions, contact us at {{admin_email}}.

Thanks,
{{site_name}}',
    'data'      => array(
        'license_id'   => $license_id,
        'license_key'  => $license_key,
        'product_id'   => $product_id,
        'product_name' => $product_name,
    ),
) );
```

---

## Context Data

When calling `kdc_qtap_send_notification()`, the `data` array becomes the context for variable resolution.

### Automatic Context Values

These are automatically available:

- `user_id` - Extracted from `recipient['user_id']` if provided
- `post_id` - From context or global `$post`
- `notification_type` - The notification type
- `notification_priority` - The notification priority

### Passing Context

```php
kdc_qtap_send_notification( array(
    'type'      => 'order_shipped',
    'recipient' => array( 'user_id' => 123 ),
    'subject'   => 'Order #{{order_number}} Shipped!',
    'message'   => 'Hi {{user_first_name}}, your order has shipped!
    
Tracking: {{tracking_number}}
Carrier: {{carrier_name}}
Estimated Delivery: {{delivery_date}}',
    'data'      => array(
        // These become available as {{variable_name}}
        'order_number'   => 'WC-12345',
        'tracking_number' => '1Z999AA10123456784',
        'carrier_name'   => 'UPS',
        'delivery_date'  => 'January 5, 2026',
        
        // These enable wp_meta lookups
        'post_id'        => $order_id,  // For {{wp_meta_postmeta_*}}
        'user_id'        => 123,        // For {{wp_meta_usermeta_*}}
    ),
) );
```

---

## Filtering Variables

### Filter Resolved WordPress Meta Values

```php
add_filter( 'kdc_qtap_notification_wp_meta_value', function( $value, $key, $context ) {
    // Mask sensitive data
    if ( $key === 'wp_meta_user_email' && ! current_user_can( 'manage_options' ) ) {
        return '***@' . substr( strrchr( $value, '@' ), 1 );
    }
    return $value;
}, 10, 3 );
```

### Filter Processed Template

```php
add_filter( 'kdc_qtap_notification_template_processed', function( $template, $context ) {
    // Add footer to all notifications
    $template .= "\n\n---\nSent via qTap Notifications";
    return $template;
}, 10, 2 );
```

### Filter Available Variables

```php
add_filter( 'kdc_qtap_notification_variables', function( $variables, $group ) {
    // Remove deprecated variables
    unset( $variables['old_variable'] );
    return $variables;
}, 10, 2 );
```

---

## Helper Functions Reference

### `kdc_qtap_process_notification_template( $template, $context )`

Process a template string with variable replacement.

**Parameters:**
- `$template` (string) - Template with `{{variable}}` placeholders
- `$context` (array) - Context data for resolution

**Returns:** string - Processed template

### `kdc_qtap_register_notification_variable( $variable_id, $args )`

Register a custom variable.

**Parameters:**
- `$variable_id` (string) - Unique identifier
- `$args` (array):
  - `label` (string) - Display name
  - `description` (string) - Help text
  - `group` (string) - Group identifier
  - `example` (string) - Example value
  - `callback` (callable) - Resolution function `function( $context )`

**Returns:** bool

### `kdc_qtap_register_notification_variable_group( $group_id, $args )`

Register a variable group.

**Parameters:**
- `$group_id` (string) - Unique identifier
- `$args` (array):
  - `label` (string) - Display name
  - `description` (string) - Group description
  - `icon` (string) - Dashicons class
  - `priority` (int) - Sort order (lower = first)

**Returns:** bool

### `kdc_qtap_get_notification_variables( $group = null )`

Get registered variables.

**Parameters:**
- `$group` (string|null) - Filter by group

**Returns:** array - Registered variables

### `kdc_qtap_notification_variables()`

Get the variables manager instance.

**Returns:** KDC_qTap_Notification_Variables

---

## Best Practices

1. **Always provide fallbacks** - Your callback should handle missing context gracefully
2. **Use descriptive names** - `order_total` not `ot`
3. **Group related variables** - Register a group for your plugin's variables
4. **Cache expensive lookups** - The system caches during a single template process
5. **Document your variables** - Include `description` and `example` in registration
6. **Respect context priority** - Check context data before falling back to database queries

---

## Changelog

### 2.0.5
- Initial release of template variables system
- Built-in Site, User, Post, Date, Notification variables
- WordPress meta prefix (`wp_meta_*`) for dynamic data access
- Child plugin variable registration API
- Variable documentation renderer for admin UI
