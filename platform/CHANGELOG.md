# Changelog

All notable changes to **kdc-qtap-platform** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.1.0] — 2026-05-01

### Added
- Initial release of the qTap Platform plugin — central tenant registry for `qtap.app`.
- Two custom post types: `qtap-tenant` (top-level directory entry) and `qtap-tenant-vertical` (hierarchical, per-(tenant, vertical) handshake state).
- `KDC_qTap_Platform_Secret` — AES-256-GCM encryption helper with `KDC_QTAP_PLATFORM_MASTER_KEY` constant override and option-based fallback.
- Custom audit table `wp_kdc_qtap_platform_audit` (insert-only, with paginated admin reader).
- Federation REST under `kdc/v1/qtap/platform/federation`:
  - `GET /tenants` — full directory listing
  - `GET /tenants/version` — cheap version probe (matches education v1.1.3's `config_version` pattern)
  - `GET /secrets/{slug}/{vertical}` — decrypt + return secret (audit-logged on every call)
- Bootstrap-secret HMAC auth (apps/web ↔ qtap.app trust) — bcrypt-hashed at rest, plaintext shown once on rotation.
- Tenant-wizard public registration endpoint `POST /qtap/platform/register` — rate-limited (5/IP/hour), creates tenant in `pending_approval`.
- Platform-initiated handshake (`KDC_qTap_Platform_Handshake::activate_vertical()`) — generates secret, POSTs to `{canonical_domain}{handshake_path}`, encrypts and stores on success.
- Secret rotation flow with HMAC-signed rotation request.
- Domain resolution helper with one-hour transient cache and version-bump invalidation; canonical-domain rename auto-appends old value to alias list.
- `directory_version` int counter bumped on every tenant/vertical save (mirrors education v1.1.3).
- Standard WordPress admin UI (no React): All Tenants list, Add New Tenant form, per-tenant edit screen with verticals table (activate/rotate/deactivate buttons), Settings page (master-key status + bootstrap-secret rotation), paginated Audit Log with action + tenant filters.
- Capability seeding (`manage_qtap_platform`, `view_qtap_platform`, plus full CPT cap set) — granted to administrator on activation, idempotent.
- Vertical registry (`kdc_qtap_platform_known_verticals` filter) with baseline entries for `education`, `events`, `finance`.

### Notes
- This plugin lives **only** on `qtap.app`. It is NOT installed on tenant WP installs.
- Events and Finance handshake endpoints are not yet exposed by the respective vertical plugins; activations for those verticals will fail until the vertical plugins ship their handshake handlers.
- `_wp_install_version` is null on every handshake until education v1.1.5 adds `plugin_version` to the handshake response.
- Outbound handshake sends the new secret in plain HTTPS — the secret IS the bootstrap material, so HMAC-signing it against itself would be circular. Existing trust-on-first-receipt pattern in education's handshake handler is the trust boundary for v0.1.0.
