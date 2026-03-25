# Changelog — kdc-qtap-admin

All notable changes to this project will be documented in this file.

## [1.0.1] — 2026-02-24

### Fixed
- REST API authentication now overrides BuddyBoss (`bb_rest_authorization_required`) and other plugins that reject cookie-less requests; `rest_authentication_errors` hook moved to priority 999 so valid CK/CS sessions clear prior plugin errors.

## [1.0.0] — 2026-02-24

### Added
- CK/CS-style REST API key generation per user (SHA-256 lookup + bcrypt verification)
- HTTP Basic Auth and query-param fallback authentication
- Sections & Elements hierarchical catalog (pre-seeded with Communication and KYC Verifications)
- Credit plans with per-element, per-direction (in/out) rates
- User credit wallets: DECIMAL(14,4) balance, plan assignment
- Atomic credit debit engine using InnoDB transactions and `SELECT FOR UPDATE`
- Credit log ledger (append-only) with balance_before/after snapshots
- REST endpoint: `POST /wp-json/kdc/v1/qtap-admin/debit`
- REST endpoint: `GET /wp-json/kdc/v1/qtap-admin/balance`
- Admin UI: Dashboard, API Keys, Sections & Elements, Credit Plans, User Credits, Credit Logs
- User deletion hook: revokes all API keys and deactivates account
- Daily WP-Cron log cleanup (default 90-day retention)
- Published hooks: `kdc_qtap_admin_loaded`, `kdc_qtap_admin_key_generated`, `kdc_qtap_admin_credits_debited`, etc.
