# Frontend Pages — Child Plugin Guide

**Available since:** kdc-qtap parent v3.1.1

When your child plugin needs a **frontend-facing page** (a published WordPress page that hosts a Gutenberg block your plugin owns), register it through the parent's frontend-pages registry instead of building your own admin UI + endpoint class. One filter call gets you four things for free:

| What you get | Where it shows |
|---|---|
| Page-picker row in **qTap App > User Dashboard** | Admins map your block to any published page or auto-create one |
| URL resolver helper (`kdc_qtap_user_dashboard()->get_frontend_page_id($id)`) | Use anywhere your plugin needs the page URL |
| Nav link in the **dashboard's elevated section** (if `visibility = rest_api`) | Surfaces below the divider for users with REST API access |
| Item in the **qTap Menu FAB** (if user matches the entry's visibility) | Floating menu shortcut |

If you don't register, your plugin is responsible for: building admin UI to pick the page, handling auto-create, resolving the URL at runtime, and contributing to the FAB / dashboard nav separately. **Use the registry — it's strictly less work.**

---

## When to use this

Register a frontend page if your child plugin owns a Gutenberg block that ends up on a public-facing WordPress page. Examples currently in production:

| Plugin | Page id | Block | Visibility |
|---|---|---|---|
| qTap Finance | `fees` | `kdc-qtap-finance/fees` | `logged_in` |
| qTap Finance | `staff` | `kdc-qtap/staff-console` | `rest_api` |
| qTap Mobile | `mobile` | `kdc-qtap-mobile/mobile-editor` | `logged_in` |
| qTap Education | `admin` | `qtap/education-dashboard` | `rest_api` |

If your block is admin-only (sits inside `wp-admin`), this isn't the right hook — use a normal admin page instead.

---

## The filter

```php
add_filter( 'kdc_qtap_frontend_pages', function ( $pages ) {
    $pages['fees'] = array(
        'label'          => __( 'Fees', 'kdc-qtap-finance' ),
        'page_label'     => __( 'Fees page', 'kdc-qtap-finance' ),
        'block_name'     => 'kdc-qtap-finance/fees',
        'auto_page_slug' => 'fees',
        'icon'           => 'wallet',
        'priority'       => 20,
        'visibility'     => 'logged_in',
        'description'    => __( 'Where logged-in users see their fees and pay invoices.', 'kdc-qtap-finance' ),
    );
    return $pages;
} );
```

### Entry fields

| Field | Required | Type | Default | Notes |
|---|---|---|---|---|
| `label` | Yes | string | — | Human display name. Shown in nav, FAB, admin row. |
| `block_name` | Recommended | string | `''` | The block id auto-inserted when admin clicks Create. Empty = placeholder paragraph. |
| `auto_page_slug` | Recommended | string | The id | Slug to look up an existing page if the admin hasn't picked one (fallback resolver). |
| `visibility` | Yes-ish | enum | `'logged_in'` | `public` / `logged_in` / `rest_api`. Drives who sees the page in nav + FAB. |
| `icon` | Recommended | string | `'layout-dashboard'` | Lucide icon name from the parent's `kdc_qtap_lucide_icons` filter. |
| `priority` | Optional | int | `50` | Sort order across all registered pages (lower = earlier in nav/FAB). |
| `page_label` | Optional | string | `"{label} page"` | Admin-row label override. |
| `description` | Optional | string | `''` | Hint shown under the admin row. |

### Page id

The array key is the **page id** — a stable slug your plugin uses to refer to this page across releases. Pick something descriptive (`fees`, `staff`, `mobile`, `tickets`). Don't change it later — admin selections persist by id.

For back-compat, the parent reserves two ids that map to pre-3.1.1 option keys:
- `staff` → `kdc_qtap_dashboard_staff_page_id`
- `admin` → `kdc_qtap_dashboard_admin_page_id`

Anything else (e.g. `fees`, `mobile`) gets the new namespace `kdc_qtap_frontend_page_id__{id}`.

### Visibility levels

```
public     — anyone, including anonymous visitors
logged_in  — any authenticated WordPress user
rest_api   — users where kdc_qtap_can_access_rest_api() is true (Staff / Admin tier)
```

The visibility controls **who sees the link**. The page itself remains accessible by direct URL — visibility is presentational, not a permission gate. Block-level access checks (e.g. "am I logged in?") still belong inside your block's render, just like before.

### Lucide icons

`icon` is a Lucide icon name keyed in the parent's `kdc_qtap_lucide_icons` filter. Common choices:

```
wallet, coins, users, smartphone, layout-dashboard, calendar, clipboard-list, bell, search
```

To add a new icon name:

```php
add_filter( 'kdc_qtap_lucide_icons', function ( $paths ) {
    $paths['ticket'] = '<path d="M2 9a3 3 0 0 1 0 6v2a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-2a3 3 0 0 1 0-6V7a2 2 0 0 0-2-2H4a2 2 0 0 0-2 2Z"/><path d="M13 5v2"/><path d="M13 17v2"/><path d="M13 11v2"/>';
    return $paths;
} );
```

The icon shows in the FAB menu, dashboard nav, and (eventually) other parent surfaces — register once, use everywhere.

---

## Reading the resolved page URL

After registration, your plugin code uses the parent helpers — **never `home_url('/fees/')` or any other hardcoded path**:

```php
$dashboard = kdc_qtap_user_dashboard();
$page_id   = $dashboard->get_frontend_page_id( 'fees' );
if ( $page_id ) {
    $url = get_permalink( $page_id );
    // Redirect, link, embed in email, etc.
}
```

The resolver tries (in order):
1. The admin's explicit selection in qTap App > User Dashboard
2. Legacy filter back-compat (only for `staff` / `admin` ids)
3. Page lookup by `auto_page_slug`
4. Returns `0` if nothing found — caller should handle this gracefully

---

## Admin auto-create flow

Each registered page gets a **+ Create new** button next to its picker dropdown. Clicking it:

1. Looks for a published page at your `auto_page_slug` and reuses it if found.
2. Otherwise creates a new published page with title = `label`, slug = `auto_page_slug`.
3. Inserts a single block of `block_name` as the page content (or a paragraph placeholder if `block_name` is empty).
4. Stores the page id in your registered option key.

Your block's render is responsible for everything from there — the parent doesn't intercept the page request.

---

## Migrating from pre-3.1.1 patterns

If your plugin previously used either of these patterns, switch to the registry:

### Pattern 1: hardcoded staff/admin filters (parent v3.0.x)

**Before:**
```php
add_filter( 'kdc_qtap_dashboard_staff_auto_page_id', function () {
    $page = get_page_by_path( 'staff' );
    return $page ? (int) $page->ID : 0;
} );
add_filter( 'kdc_qtap_dashboard_staff_block_name', fn() => 'kdc-qtap/staff-console' );
```

**After:**
```php
add_filter( 'kdc_qtap_frontend_pages', function ( $pages ) {
    $pages['staff'] = array(
        'label'          => __( 'Staff', 'kdc-qtap-finance' ),
        'block_name'     => 'kdc-qtap/staff-console',
        'auto_page_slug' => 'staff',
        'icon'           => 'users',
        'visibility'     => 'rest_api',
        'priority'       => 80,
    );
    return $pages;
} );
```

Drop the legacy filter calls — the registry replaces both. (The parent still honors them as a fallback for one or two transition releases, but new code shouldn't add them.)

### Pattern 2: dedicated endpoint class (Finance Fees, Mobile Mobile)

If your plugin maintains its own `MyPlugin_Endpoint::get_page_url()` method that's just a thin wrapper around an option lookup, replace the option with a registry registration and have your endpoint class delegate to `kdc_qtap_user_dashboard()->get_frontend_page_id( 'my-page' )`. The endpoint class can stay as a public API — it just stops owning the storage.

```php
// In your endpoint class:
public function get_page_url() {
    $page_id = function_exists( 'kdc_qtap_user_dashboard' )
        ? kdc_qtap_user_dashboard()->get_frontend_page_id( 'my-page' )
        : 0;
    return $page_id ? get_permalink( $page_id ) : '';
}
```

This lets your existing callers keep working while moving the storage + admin UI to the centralized location.

---

## Visibility-level guidance

| Use case | Visibility |
|---|---|
| User views their own data (fees, mobile numbers, profile) | `logged_in` |
| Storefront-style page with sign-up gate inside the block | `logged_in` |
| Staff-only console with REST API tooling | `rest_api` |
| Admin-tier dashboard | `rest_api` |
| Anonymous-accessible landing page | `public` |

The visibility doesn't lock the URL — it controls whether the link **surfaces** in qTap nav and FAB. If a non-elevated user navigates directly to a `rest_api`-visibility page, your block's render is still what enforces access. Treat visibility as a UX hint, not a security boundary.

---

## Common gotchas

- **The id in the array key is permanent.** Renaming it from `staff` to `staff-console` later orphans every admin's existing selection. Pick a stable id up front.
- **Block name must match the registered block exactly** (e.g. `kdc-qtap-finance/fees`, not `qtap-finance/fees` or `fees`). The auto-create writes `<!-- wp:{block_name} /-->` literally.
- **`auto_page_slug` collisions.** If two child plugins both want the slug `dashboard`, the second one to register wins for the auto-detect lookup. Use plugin-prefixed slugs when ambiguity is possible (`finance-fees`, not `fees`, when you anticipate other plugins might also want a `fees` slug — though in practice plugin domains are usually unique enough that this rarely matters).
- **Don't hook the filter conditionally on plugin version.** If your child plugin runs against an older parent that doesn't know about `kdc_qtap_frontend_pages`, the filter just never fires — no error. Your registration is a no-op on parent < 3.1.1.
- **The icon must be registered.** If you specify `'icon' => 'my-custom-icon'` and don't add it to `kdc_qtap_lucide_icons`, the FAB falls back to the layout-dashboard glyph. Register the icon path alongside the page registration.

---

## End-to-end example: a hypothetical Tickets plugin

```php
// 1. Add the icon shape (parent's Lucide registry).
add_filter( 'kdc_qtap_lucide_icons', function ( $paths ) {
    $paths['ticket'] = '<path d="M2 9a3 3 0 0 1 0 6v2a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2v-2a3 3 0 0 1 0-6V7a2 2 0 0 0-2-2H4a2 2 0 0 0-2 2Z"/><path d="M13 5v2"/><path d="M13 17v2"/><path d="M13 11v2"/>';
    return $paths;
} );

// 2. Register the frontend page.
add_filter( 'kdc_qtap_frontend_pages', function ( $pages ) {
    $pages['tickets'] = array(
        'label'          => __( 'My Tickets', 'kdc-qtap-events' ),
        'page_label'     => __( 'My Tickets page', 'kdc-qtap-events' ),
        'block_name'     => 'kdc-qtap-events/my-tickets',
        'auto_page_slug' => 'my-tickets',
        'icon'           => 'ticket',
        'priority'       => 30,
        'visibility'     => 'logged_in',
        'description'    => __( 'Where attendees view their purchased tickets.', 'kdc-qtap-events' ),
    );
    return $pages;
} );

// 3. Use the resolver wherever your plugin needs the URL.
function kdc_qtap_events_my_tickets_url() {
    if ( ! function_exists( 'kdc_qtap_user_dashboard' ) ) {
        return '';
    }
    $page_id = kdc_qtap_user_dashboard()->get_frontend_page_id( 'tickets' );
    return $page_id ? get_permalink( $page_id ) : '';
}
```

That's everything. After activation, the admin sees a "My Tickets page" row under qTap App > User Dashboard > Frontend pages > Logged-in users, with a + Create new button. Logged-in users see the link in the qTap Menu FAB. URL resolution, page-id storage, auto-create, FAB rendering — all handled by the parent.
