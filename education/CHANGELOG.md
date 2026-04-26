# qTap Education — Changelog

All notable changes to this plugin will be documented here.

## [1.0.49] — 2026-04-26

### Changed
- **"WordPress user email" question is no longer asked when only one side is allowed.** Previously the radio (Use Parent address / Use Student address) showed regardless of the Allow Parent / Allow Student state — even though the question is moot when only one side is enabled. New behavior:
  - **Both** Allow Parent + Allow Student checked → radio appears, the saved choice wins.
  - **Only one** of them checked → radio is hidden; `wp_user_email_source()` auto-resolves to the active side (saved value ignored).
  - **Neither** checked → radio hidden; falls back to `'parent'` (legacy default).
- The hide/show logic is reactive: ticking either Allow checkbox updates the row's visibility immediately (plain JS in the General tab).
- Description copy under the radio updated to clarify the both-allowed-only semantics.

### Fixed
- **Server-side resolver matches the UI rule.** `KDC_qTap_Education_Google_Workspace::wp_user_email_source()` now short-circuits to the active side when only one Allow flag is true — so the WP `user_email` constructed in REST `create_student` always uses a suffix that the admin actually has enabled, even if the saved radio value is stale.

## [1.0.48] — 2026-04-26

### Fixed
- **Collect-mode email field no longer leaks into Assign mode.** The "WordPress user email *" input rendered with the HTML `hidden` attribute, but `.qtap-education-dashboard__field { display: flex; }` (a class selector) outranked the user-agent `[hidden] { display: none; }` rule, so the field stayed visible regardless of mode. Added `.qtap-education-dashboard__field[hidden] { display: none; }` to restore the expected behavior.
- **Username override now updates the main "Available / Taken" badge** when re-checked. Previously, even after staff entered an alternative (e.g. `zuvi.kudmule1`) and clicked Check → "✓ Available", the main row still showed `⚠ Taken` (because that badge was bound to the *original* generated username's check). Now an "Available" override result swaps the main badge to "✓ Available" — the badge reflects what will actually be created on submit, not what was originally generated. Same swap for "Still taken" / "Invalid format" → main badge stays "⚠ Taken".

### Changed
- **Conflict status text adds a "this will be used when you submit" hint** so staff aren't left guessing whether they need a separate "Apply" action — they don't; just submitting the form is enough. New label `willBeUsed` localizable in render.php.

## [1.0.47] — 2026-04-26

### Fixed
- **Create form preview now respects the configured Email domain.** The `siteHost` value localized to view.js was hardcoded to `wp_parse_url( home_url(), PHP_URL_HOST )` (the WP site host), so the Parent / Student preview rows kept showing e.g. `…@tridha.edu.in` even after the admin set Email domain to `tridha.com` in Education > Settings > General > Email. Now the preview reads through `KDC_qTap_Education_Google_Workspace::general_email_domain()` — same lookup chain as the REST handler that creates the WP user — so the preview, the WP user_email, and (when GWS is active) the Workspace account address all use the same domain.
- **Username override now drives the Parent / Student email preview.** Previously, when a username collision triggered the "Username already taken — pick another" alert, staff could type an alternative (e.g. `zuvi.kudmule1`) and Check it, but the Parent Account / Student Account preview rows kept showing the *old* taken username (`zuvi.kudmule_parent@…`). The preview now updates live as the staff types the override — so what the staff sees is exactly what gets created (`zuvi.kudmule1_parent@…`). Server-side already used the override for both the WP `user_login` and the email's local-part; this is purely a UI sync fix.

## [1.0.46] — 2026-04-26

### Added
- **"Email provider" dropdown** as the second question in Education > Settings > General > Email (right after Primary email handling). Choices:
  - **Manual** (default) — no external provisioning. Only WP user records are created when staff submits the Create form.
  - **Google Workspace** — routes Parent / Student account creation through the GWS adapter (configured in the GWS tab).
  - List items below "Manual" sort alphabetically; future providers (Microsoft 365, Zoho, etc.) will slot into that order.
- **GWS adapter helper `email_provider()`** — single source of truth for any consumer that needs to know the active provider slug. Whitelisted to known values; unknown values fall back to `'manual'`.

### Changed
- **WordPress user email row moved to the bottom of the Email section.** The radio sample addresses (`first.last_parent@…`, `first.last@…`) are derived from the Parent/Student suffixes + Email domain set above, so showing this row last makes the cause-and-effect obvious to admins.
- **Provider-neutral radio labels.** Dropped "Google Account" hardcoding from the WP user email options:
  - "Use the Parent Google Account address (…)" → "Use the Parent address (…)"
  - "Use the Student Google Account address (…)" → "Use the Student address (…)"
- **Description copy** for the WP user email row now clarifies that the sample addresses are computed from the suffixes + domain set in the same section, and references "Create … Account" (provider-neutral) instead of "Create … Google Account".

### Note
- Provider routing is captured but not yet wired to provisioning behavior — selecting "Manual" doesn't currently disable Workspace provisioning when the GWS tab's own enable-toggle is on. Wiring the dropdown to be the master switch (and removing redundancy with the GWS-tab toggle) is a follow-up; for this release the field is metadata + future-proofing.

## [1.0.45] — 2026-04-26

### Changed
- **Provider-neutral labels.** Dropped "Google Account" from the Create form labels — preview rows now read "Parent Account" / "Student Account" and toggles read "Create Parent Account" / "Create Student Account". The label text is provider-neutral so the same UI can host Google, Microsoft 365, Zoho, etc. in future without copy churn.
- **Email settings restructured under a new "Email" section in the General tab.** Moved `parent_email_suffix` + `student_email_suffix` out of the GWS tab and into General — they're account-creation policy, not Workspace-specific config. The GWS tab now only houses Workspace-specific things (provisioning toggle, domain, super-admin, service-account JSON, OU paths, force-password-change).
- **REST `create_student` enforces the new Allow gates server-side.** Even if a client POSTs `create_parent_google=true`, it's forced false when "Allow Parent account" is unchecked. Same for student.

### Added
- **"Allow Parent account" + "Allow Student account" toggles** in General > Email. Each reveals its own suffix field when ticked. Defaults: parent ON, student OFF (preserves prior Create-form behavior). Disabling either hides the matching toggle on the Create form *and* short-circuits Workspace provisioning at the REST level.
- **GWS adapter helpers**: `allow_parent_account()`, `allow_student_account()`, plus `parent_email_suffix()` / `student_email_suffix()` now read from the new `general[]` location first with a back-compat fallback to the legacy `gws[]` location (no migration needed).

### Fixed
- **No more phantom "Will create" badge when GWS is disabled.** Previously the badge logic defaulted to "checked" when the underlying checkbox didn't exist in the DOM, so the preview rows showed "✓ Will create" even though no provisioning would happen. Now the badge is suppressed entirely (tri-state: `true`/`false`/`null = no badge`) when the matching toggle is absent.

### Note
- Existing sites: suffixes saved on <= v1.0.44 stay readable via the back-compat fallback. The GWS tab's save handler carries the legacy values forward so the fallback keeps resolving until the admin re-saves the General tab.

## [1.0.44] — 2026-04-26

### Fixed
- **Save button now stays visible after unticking "Enable Google Workspace user provisioning".** Previously the entire integration-fields wrapper (including the Save button at its bottom) was hidden via `display:none` when the checkbox was unticked, leaving the admin no way to persist the disable. The Save button now lives outside the hidden wrapper.
- **Disabling integration no longer wipes existing data.** When the admin unticks the integration and saves, the existing `domain`, `super_admin_email`, OU paths, email suffixes, service-account JSON, and verification status all stay in the DB — only the `enabled` flag flips. Re-enabling later brings the configuration right back without re-entering credentials. Implemented via per-field fallback to existing values in the save handler.

### Note
- Behavior when GWS is disabled: the *Create Parent / Student Google Account* toggles + *Send login to Contacts* toggle on the Create form are replaced with a "Google Workspace not configured" note. WP user records are still created with the auto-generated email pattern (Assign mode) or a staff-collected email (Collect mode); no Workspace API calls fire and no welcome notifications dispatch (those depend on successful Workspace creation).

## [1.0.43] — 2026-04-26

### Added
- **"Primary email handling" mode** in Education > Settings > General — radio with two options:
  - **Assign — auto-generate** (default) — current behavior; the WP `user_email` is generated from `{login}{suffix}@{domain}` driven by the WP-user-email-source + Email-domain settings.
  - **Collect — staff enters an external email at create time** — for non-institute deployments where students bring their own email (Gmail, Outlook, etc.).
- **Collect-mode field on the Create form**: when mode = collect, a `WordPress user email *` input appears below the Identity grid; the auto-generated parent/student email preview rows are hidden. The username preview row stays (it still drives `user_login`).
- **REST `create_student` validates collected emails server-side**: required when mode = collect; must pass `is_email()`; rejects with `qtap_edu_user_email_taken` (409) when the address already belongs to a WP user.
- **Settings page hides the auto-generation rows** ("WordPress user email" radio + "Email domain") when Collect mode is selected — they're irrelevant in that mode. Plain JS, no jQuery dep.
- **GWS adapter helper `email_mode()`** reads `general[email_mode]` from the option, defaulting to `assign`.

### Note
- Drag-to-reorder priority list of email patterns is intentionally still deferred. Between Parent and Student the existing radio already expresses "primary"; a real reorder primitive only earns its keep when a third pattern is added. Easy to bolt on later if you grow more patterns.

## [1.0.42] — 2026-04-26

### Changed
- **"WordPress user email" setting moved from Google Workspace tab to General tab.** It belongs there: the email source is account-creation policy that applies regardless of whether Workspace integration is enabled. The previous storage location (`gws[wp_user_email_source]`) is still read as a back-compat fallback — no migration required for sites saved on v1.0.40/v1.0.41.

### Added
- **Email domain field** in Education > Settings > General. Controls the domain inserted after "@" in generated WP user emails:
  - **Google Workspace active+verified** → field is read-only and mirrors the Workspace primary domain (Google manages DNS/MX, so it's the source of truth). A green-checked notice tells the admin to change it in the GWS tab.
  - **Workspace disabled or not yet verified** → admin owns the field; defaults to the WP site host.
  - The setting is stored under `kdc_qtap_education_settings['general']['email_domain']`.
- **REST `create_student`** now resolves the WP `user_email` domain via `KDC_qTap_Education_Google_Workspace::general_email_domain()` (new helper) instead of always using `wp_parse_url( home_url(), PHP_URL_HOST )`. The helper prefers the live GWS domain when active+verified, falls back to the admin's `general[email_domain]` value, then the WP site host.

### Note
- The "Assign vs Collect" mode + drag-to-reorder pattern list is intentionally deferred to a follow-up release once a real "non-institute / collect external email" deployment exists to validate against. This release covers the move-to-General + domain-field portion only.

## [1.0.41] — 2026-04-26

### Note
- Internal release: refinements to `KDC_qTap_Education_Notifications::register_types()` (added `name`, `icon`, `audience`, `supported_channels` keys per the parent's richer registration shape).

## [1.0.40] — 2026-04-26

### Added
- **Admin setting: "WordPress user email" source.** New radio in **Education > Settings > Google Workspace**, between the email-suffix fields and the force-password-change toggle. Lets the admin pick which generated address fills the WP `user_email` field for newly-created students:
  - **Parent Google Account address** (`first.last{parent_suffix}@domain`) — default; preserves prior behavior.
  - **Student Google Account address** (`first.last{student_suffix}@domain`).
- The selected address always fills WP `user_email` regardless of whether *Create Parent / Student Google Account* is ticked on the Create form. The toggles only control Workspace provisioning; the WP record always carries the chosen address (WP requires `user_email` to be non-empty).
- **Works even when Google Workspace integration is disabled.** The suffix fields are stored independent of the integration's `enabled` flag, so admins running without Workspace can still configure the email pattern (e.g. set the parent suffix to `_parent` and the student suffix to `.student` and pick which side drives WP).

### Changed
- REST `create_student` resolves `user_email` via `KDC_qTap_Education_Google_Workspace::wp_user_email_suffix()` (new helper) instead of always using `parent_email_suffix()`. The helper reads the new `wp_user_email_source` setting and returns the matching suffix.

### Note
- No migration on existing users — only newly-created students from v1.0.40 onward respect the new source setting. Sites that haven't opened the GWS settings tab default to `'parent'` (matching v1.0.39 behavior).

## [1.0.39] — 2026-04-26

### Changed
- **WP `user_email` now derived from the configured Parent email suffix.** WordPress requires every user record to have an email; the qTap Education create flow always uses the Parent Google Account address pattern (`{username}{parent_suffix}@{site_host}`) for `user_email` regardless of whether the *Create Parent Google Account* checkbox is ticked. Two practical effects:
  - The WP user_email now matches the value shown in the Create form's "Parent Google Account" preview row, even when the admin opts to skip Workspace provisioning. Previously the WP user_email was hardcoded to `{username}_parent@…` while the preview reflected the configured suffix — they could disagree if an admin had changed the suffix in Education > Settings > Google Workspace.
  - Admin changes to **Parent email suffix** now flow through to all newly-created students' WP user_email automatically.
- The "Create Parent Google Account" checkbox now controls only Workspace provisioning (whether the account is created via Admin SDK), not whether the WP user record carries that address. The WP record always carries it.

### Note
- No migration on existing users — only newly-created students from v1.0.39 onward use the configured suffix. Sites that never customized the suffix (default `_parent`) see no change at all.

## [1.0.38] — 2026-04-26

### Changed
- **Parent / Student Google Account preview rows now show a "Will create" / "Will skip" badge.** Previously the rows were hidden when the corresponding checkbox was unticked, which left the admin without a visual cue about what the generated email *would* be. Now both rows stay visible regardless of checkbox state, and a badge — modeled on the existing username `✓ Available` badge — sits next to each preview value:
  - **Parent ticked** / **Student ticked** → green `✓ Will create` (matches the success-state of the username availability badge for visual continuity).
  - **Parent unticked** / **Student unticked** → muted italic `Will skip`, with the email value also strike-through + dimmed (`opacity: 0.55`) so the row reads as informational-only.
- The badge updates live as the staff toggles either checkbox; the row's `is-skip` class is the single CSS hook for the muting effect.

## [1.0.37] — 2026-04-26

### Changed
- **Welcome + credentials notifications now align with parent qTap App v2.7.9 conventions.** The two notification types (`qtap_education_welcome_new_user` + `qtap_education_login_credentials`) are wired through the parent's full registration recipe so admins can edit the templates centrally in **qTap App > Notifications > Templates** and see them attributed to Education in the Source filter / summary card.
  - **Default templates** are now registered via `kdc_qtap_default_notification_templates` filter — single source of truth, no longer duplicated inline in every send call.
  - **Template lookup at send time** uses `kdc_qtap_notifications()->get_template($type)` so admin-edited templates win automatically (lookup chain: customized → registered defaults → empty). Pre-2.7.9 parents fall back to our local default registration so the plugin keeps working unchanged.
  - **WhatsApp template config** (template_name|lang, header, body, footer, buttons) is also picked up from the admin-edited template — admins who customize the WhatsApp tab in parent's editor get those values forwarded into the `whatsapp_*` send args.
  - **Variable palette** registered via `kdc_qtap_register_notification_variables` action. The Templates editor's variable palette now lists Education-specific variables (`{{parent_email}}`, `{{parent_login}}`, `{{parent_password}}`, `{{parent_login_url}}`, `{{student_email}}`, `{{student_login}}`, `{{student_password}}`, `{{student_login_url}}`, `{{contact_name}}`, `{{login_url}}`, `{{app_url}}`) under a new "Education" group.
  - **Type ownership** declared via `kdc_qtap_notification_type_owners` filter (REQUIRED as of v2.7.9) so the parent's Source filter, summary card, and Edit Template deep links can attribute the two types to `kdc-qtap-education`.

### Note
- No behavioral change for sites that have never edited templates — the registered defaults match the previous inline copy verbatim, so existing welcome / credentials notifications keep going out unchanged.

## [1.0.36] — 2026-04-26

### Changed
- **Search input now matches the form field font size exactly.** Previously the "Type a name…" input was anchored to `var(--kdc-qtap-font-size-lg)` (1.125rem) while the First Name / Last Name fields used `1em`, which inherits through the wrapper's 1.125em scale. The two values were nominally equal but rendered subtly differently because one is rem-anchored and the other is em-relative — so glyphs in the search row appeared a hair thinner/smaller than the form below. Switching the search to `font-size: 1em` puts both on the same em ladder; identical glyph metrics. Min-height stays 44px (matching the lg buttons sharing the row), padding tweaked to em units (`0.55em 0.8em`) to scale with the new font baseline.

## [1.0.35] — 2026-04-26

### Added
- **Dynamic two-row Google Workspace preview.** The account-preview panel inside the Create form now shows a row per requested Workspace account, gated on the corresponding checkbox:
  - "Parent Google Account" row appears only when *Create Parent Google Account* is ticked, and uses the **parent_email_suffix** from Education → Settings → Google Workspace.
  - "Student Google Account" row appears only when *Create Student Google Account* is ticked, and uses the **student_email_suffix** from the same settings.
  - Suffixes are now read from settings (no longer hardcoded `_parent`), so changing them in admin reflects in the preview without code edits.
- **Silent Title-Case for First Name / Last Name.** On `blur`, e.g. `john  KEMP` is rewritten to `John Kemp` — done on blur (not input) so we don't fight the user mid-typing. The preview updates to match.

### Changed
- **Search row sizing.** "Type a name…" input + "All fields" select now match the height of the lg buttons next to them on the same row (`min-height: 44px`, `font-size: var(--kdc-qtap-font-size-lg)`, `padding: 12px 16px`).
- **Student personal email field sizing.** Now mirrors the pill-toggle's metrics (1.5px border, 0.6em radius, 0.7em/0.9em padding) so the email input and the *Create Student Google Account* pill on the row above appear as same-height twins.
- **Contact-row name placeholder** updated from `Name (e.g. Father)` to `Name (e.g. Mother)`.

## [1.0.34] — 2026-04-25

### Added
- **Shareable tab links + Create-new deep link.** The top-level Count / Student tabs are now URL-addressable, so admins can copy the address bar and share. URL slugs are vertical-agnostic — `?tab=count` and `?tab=user` (the `user` slug stays meaningful when the same dashboard is reused for non-school verticals; internal radio keys `count` / `student` are unchanged for back-compat with existing CSS).
  - `?tab=count` → activates the Count section on load.
  - `?tab=user` → activates the Student section on load.
  - `?tab=user&action=create` → activates Student AND opens the Create New form.
  - Tab clicks update the URL via `history.replaceState` (no extra browser-history entries, no scroll).
  - Toggling the Create form open/closed adds/removes `&action=create` automatically.
- **Routing layer is JS-only** (`initUrlRouting()` in `view.js`) — no PHP changes, no CSS changes, no migration. The radio-driven tab system continues to work on JS-disabled pages; the URL routing is progressive enhancement.

### Note
- Slug-key map: `{ count: 'count', student: 'user' }`. To add a new top-level section in the future, add an entry to both `KEY_TO_SLUG` and `SLUG_TO_KEY` maps in `view.js`'s `initUrlRouting()`.

## [1.0.33] — 2026-04-25

### Changed
- **Login-required state now shows the inline `wp_login_form()`** instead of a "Please log in" sentence + Log in link. When a logged-out visitor hits the Education Dashboard block they get a real username/password form right inline, with a Remember-me checkbox and a redirect_to back to the same page on success. Form is styled to match the dashboard's input/button scale (font: inherit, line-height 1.5, primary-color submit button). The form ID `qtap-education-dashboard-loginform` is what the CSS targets — any plugin that filters `login_form_*` (Login With OTP, Google Sign-In, etc.) still works because `wp_login_form()` runs through the standard WP filters.

## [1.0.32] — 2026-04-25

### Added
- **Student's Personal Email field** on the Create form. New input under the Student toggle, hidden until *Create Student Google Account* is ticked, then required (HTML5 `required` + `inputmode="email"` + format validation client-side AND server-side). Persisted to `kdc_qtap_education_student_personal_email` user_meta. This is the single external destination for student-account credentials — we explicitly do **not** collect a student mobile.
- **Welcome + credentials notifications now also dispatch to the student's personal email** when a student account was created and an email is on file. Email-only channel (no WhatsApp/SMS — we have no student mobile). Parent contacts continue to receive the parent-account credentials separately. Two distinct dispatches, both flowing through parent's notification framework with the existing `qtap_education_*` types.

### Changed
- **Toggle layout reshaped to 2×2 grid.** Row 1: *Create Parent Google Account* | *Send login to Contacts*. Row 2: *Create Student Google Account* | Student's Personal Email input. The email cell collapses to empty when Student is unticked. Stacks to single column on viewports ≤600px.
- **Section title legends bumped** for IDENTITY / CONTACTS / ENROLLMENT — now 1em (was 0.85em), with a 2px primary-color bottom border and a small accent dot before the label so each fieldset reads clearly as the start of a new section. Spacing between sections increased.
- **Search input visually matches the Create form fields.** Added `font: inherit` + `line-height: 1.5` to the search-row override so Finance's `font-size: 13px` (and any inherited line-height differences) are fully neutralized — height + font now line up with First Name / Last Name.
- **Empty associated-user "selected" container is hidden.** The faint placeholder block under the sibling search no longer shows when no sibling is picked.

### Note
- "Why is the search input ID `qtap-edu-1-...`?" — the `1` is from WordPress's `wp_unique_id('qtap-edu-')` which auto-increments per block instance on the page. It's not configurable, just a unique-ID counter; second instance on the same page would be `qtap-edu-2-...`. Cosmetic only — has no functional impact.
- `Academic Year`, `Grade`, and `Division` already use Finance's label helpers (`kdc_qtap_finance_label('academic_year')` etc.) via `KDC_qTap_Education_Dashboard::label()`. Admin-customized labels in Finance flow through automatically — has been so since v1.0.0.

## [1.0.31] — 2026-04-25


### Fixed
- **"Send login to Contacts" no longer enabled when no Workspace account is being created.** v1.0.30 split the single "Create Google Account" toggle into Parent + Student but didn't carry the dependency rule from v1.0.x — when both new toggles were unticked, "Send login to Contacts" stayed enabled, which is illogical (nothing to send). The dependency now watches BOTH new checkboxes (plus the legacy `create_google_account` for older renders): Send login is enabled if EITHER target is ticked, and disabled + auto-unticked when both are off.

## [1.0.30] — 2026-04-25

### Added
- **Parent + Student Workspace targets** — independent accounts. Each new student can now optionally provision **two** Google Workspace accounts:
  - **Parent account** (default ON, ticked) — for the parent's portal login, fee payments, etc. Email: `first.last_parent@domain` (suffix configurable).
  - **Student account** (default OFF, unticked) — for the student's own Workspace (Classroom, school email). Email: `first.last@domain` (suffix configurable, default empty).
  - Admin can tick/untick either checkbox per student.
  - Each account is created in its own configurable OU (Parent OU + Student OU in *qTap Education > Settings > Google Workspace*).

- **Two OU paths in settings.** Replaces the single OU field with `Parent OU path` + `Student OU path` (each separately validated by Test Connection). Pre-v1.0.30 single `ou_path` is read as the parent OU for upgrades.

- **Two email-suffix fields in settings** — `parent_email_suffix` (default `_parent`) and `student_email_suffix` (default empty), so admins can shape both email patterns to match school conventions.

- **Per-target user_meta keys**: `kdc_qtap_education_workspace_parent_*` and `kdc_qtap_education_workspace_student_*` (status, user_id, email, created, error, suspended). Legacy `workspace_*` keys still mirror the parent target for back-compat with existing UI / reports.

- **Combined notification dispatch.** Welcome and credentials notifications now fire **once** per student create — from the LAST scheduled job to settle, so the message can include credentials for both targets in one go. The completion barrier looks at the OTHER target's status (`pending`/`processing` = wait; anything else = fire). Notifications go to **parent contacts only** (no student external contact data is collected). Templates extended with `{{parent_email}}`, `{{parent_login}}`, `{{parent_password}}`, `{{student_email}}`, `{{student_login}}`, `{{student_password}}`. Missing target's variables resolve to empty strings — admin can edit templates in *qTap > Notifications*.

- **User-edit metabox split.** The "Google Workspace" section on `user-edit.php` now shows TWO subsections (Parent / Student), each with its own status + lifecycle dropdown. Admin can suspend the student account while leaving the parent active, etc. Multiple per-target actions on one save show all results as separate admin notices.

### Changed
- **`KDC_qTap_Education_Job_Handler::schedule()`** signature extended: 4th arg is an optional `array $args = ['target','email','ou_path']`. The 3-arg call shape (legacy) still works and defaults to `target=parent`.
- **`KDC_qTap_Education_Notifications::send_credentials()`** signature: prefers the new 2-arg form `($user_id, $password)` which sources both targets' details from user_meta. The 4-arg legacy form still works for the inline dispatch path.
- **REST `POST /students`** accepts `create_parent_google` (default true) + `create_student_google` (default false). Legacy `create_google_account=true` is honored as `create_parent_google=true`. Response now includes `workspace_parent_scheduled` + `workspace_student_scheduled`.
- **REST `POST /google-workspace/test-connection`** now validates BOTH the parent and student OU paths (when provided). `verified` flips on only when all probes pass. Response shape switched from `ou_check` (single object) to `ou_checks` (array, one per probed path).
- **Frontend Create form** now shows three pill toggles: *Create Parent Google Account*, *Create Student Google Account*, *Send login to Contacts*.

### Architecture notes
- **One seed password serves both targets.** Both Workspace accounts force-change-on-first-login per the integration's standing setting; the user picks their own password the first time they sign in to either.
- **Failure isolation.** If parent succeeds and student fails (or vice versa), notifications still fire, mentioning whichever account exists. Failed target shows in the user-edit metabox with a Retry option.
- **No separate "send to student" path.** The student doesn't receive a separate notification — all credentials go to the parent's contact rows. Architecturally simpler and matches the data we collect.

## [1.0.29] — 2026-04-25

### Added
- **OU path validation at Test Connection.** The Test Connection button now also probes `GET /admin/directory/v1/customer/my_customer/orgunits/<path>` to confirm the configured OU actually exists before save. If the OU is missing, the panel shows `⚠ OU check failed: OU path "..." does not exist...` with a hint about including parent OUs (e.g. `/Indian Education Revival Trust (IERT)/Tridha Parents Group`). When the OU check fails, `verified` stays `false` so the Create form's checkbox stays gated. Catches the most common Workspace setup mistake before the first student is created.
- **Public `verify_ou_exists()`** method on the GWS adapter so REST handlers, job handlers, or future user-edit flows can validate an OU path without re-running the full Test Connection.
- **`SCOPE_ORGUNIT_RO` constant** for future read-only OU listing (the dropdown picker is the planned follow-up).

### Changed
- **Auto-prefix `/` on OU path save.** If an admin enters `Tridha Parents Group` (without leading slash, as the form lets them), the save handler now silently normalizes it to `/Tridha Parents Group`. Empty stays `/`. Trailing slashes are trimmed. Same normalization is applied in the REST test-connection handler so the live test runs against the canonical form.
- **Path-segment URL encoding** in the OU verify call: each segment is `rawurlencode`'d individually so spaces and parens (e.g. `Indian Education Revival Trust (IERT)`) survive, but slashes between segments stay unencoded (Google's API uses them as URL path separators).

### Note
- The auto-prefix only fixes the **leading slash** mistake. The other common mistake — missing **parent OU** in the path (e.g. entering `/Tridha Parents Group` when the OU is actually nested under `/Indian Education Revival Trust (IERT)`) — is now caught by the new OU-exists check at Test Connection time, with a specific error message pointing the admin to the full path. The OU dropdown picker (deferred to a future release) will eliminate both classes of error entirely.

## [1.0.28] — 2026-04-25

### Added
- **Google Workspace lifecycle controls on the WP user-edit screen.** New "Google Workspace" section appears on user-edit / profile screens for any user with a `kdc_qtap_education_workspace_status` value. State-aware UI:
  - **`created`**: shows status header (Workspace user ID, created-at) plus a dropdown — *No change* (default) / *Activate* / *Suspend* / *Delete*. Save runs the corresponding Admin SDK call inline; admin gets a one-shot `admin_notices` flash with the result. Delete uses a JS `confirm()` because Google has no soft-delete.
  - **`pending` / `processing`**: shows the status; no actions (job is in flight or about to be).
  - **`failed:<code>`**: shows status + the verbatim error message. If the seed password is still on file, also shows a "Retry Workspace creation on save" checkbox that re-schedules the creation job.
  - **No status meta at all**: section is omitted (keeps user-edit tidy for non-Education users).
- Three new methods on `KDC_qTap_Education_Google_Workspace`: `activate_user($id)`, `suspend_user($id)`, `delete_user($id)`. Activate/Suspend `PATCH /admin/directory/v1/users/<id>` with `{ suspended: false|true }`; Delete `DELETE /users/<id>`.

### Changed
- On Activate/Suspend, a new `kdc_qtap_education_workspace_suspended` user_meta is set (1 = suspended, 0 = active) so reports/filters can quickly find suspended accounts without round-tripping to Google.
- On Delete, `workspace_status` is set to `deleted` and the Workspace user ID / created-at / error metas are wiped. Status survives so admins can see the account was once provisioned.

### Notes
- Lifecycle actions run inline (not via the cron-scheduled job) because the admin is staring at the screen and expects immediate feedback. Failures surface in the admin notice.
- Delete is **irreversible** — Google's API doesn't expose a soft-delete. The JS confirm() prompt is the only guardrail; consider a stronger confirmation pattern (type DELETE) in a later release if accidental deletions become an issue.

## [1.0.27] — 2026-04-25

### Added
- **Google Workspace user provisioning.** When a school configures the integration in *qTap Education > Settings > Google Workspace*, ticking "Create Google Account" on the Create Student form now actually creates a user in the school's Workspace via the Admin SDK. The flow:
  1. Staff submits the form. WP user is created (fast); a background job is scheduled and the response returns immediately.
  2. ~15 seconds later (or on the next admin page load if WP-Cron is sleepy), the job runs. Hand-rolled service-account JWT auth (RS256 via `openssl_sign`) exchanges for an OAuth access token, then `POST /admin/directory/v1/users` creates the Workspace user in the configured OU with `changePasswordAtNextLogin: true`.
  3. On success: user_meta `kdc_qtap_education_workspace_status=created`, plus the welcome notification fires (always) and credentials notification fires (only if "Send login to Contacts" was ticked).
  4. On failure: status is set to `failed:<code>` with the verbatim error message; staff can retry from the user-edit screen (next release).
- **Settings page with verbose setup guide.** New `qTap Education > Settings` page has tabs (General + Google Workspace). The Workspace tab includes a master toggle + reveal pattern, fields for domain / super-admin email / service-account JSON upload / OU path / force-password-change toggle, and a step-by-step setup guide with direct links to Google Cloud Console (Admin SDK enablement, Service Account creation) and Workspace Admin (Domain-wide Delegation page). Test Connection button probes the credentials inline before save.
- **Notification types registered with parent.** Two new types via `kdc_qtap_register_notification_type()`:
  - `qtap_education_welcome_new_user` — sent to all contacts after Workspace creation succeeds. Default channels: email + WhatsApp. Variables: `{{site_name}}`, `{{user_name}}`, `{{contact_name}}`, `{{login_url}}`, `{{app_url}}`.
  - `qtap_education_login_credentials` — sent to all contacts when "Send login to Contacts" is ticked, **only after Workspace creation succeeds**. Variables include `{{login}}`, `{{email}}`, `{{password}}`. Templates editable in *qTap > Notifications*.
- **AES-256-GCM encryption helper** (`KDC_qTap_Education_Secret`) for the service-account JSON at rest, keyed off WP `AUTH_KEY`. The encrypted blob is stored in the `kdc_qtap_education_settings` option; decrypted only at API-call time.
- **`kdc-qtap-frontend-helpers` viewScript dep** (carried over from v1.0.26) — already explicit in the block registration.

### Changed
- **"Create Google Account" toggle is gated.** When the Workspace integration is not enabled+verified in settings, the toggle is hidden and replaced with a one-line note pointing the staff (admin: with link, non-admin: with message) to *Settings > Google Workspace*. Prevents collecting opt-in for an integration that won't fire.
- **`send_login_to_contacts` timing.** When Workspace integration is active, the credentials notification now waits until Workspace creation succeeds (so the credentials emailed actually work). When not active, falls back to inline dispatch as before.
- **Welcome + credentials notifications now use parent's framework.** Templates are admin-editable in *qTap > Notifications* and respect channel-enabled flags.

### Fixed
- **`$login_result` undefined warning** in the create-student response when `username_override` was supplied (introduced in v1.0.22). Now uses `isset()` guard. The bug was harmless in practice (the override path returns null for `username_taken_suffix` anyway) but PHP would emit a notice.

### Architecture notes
- Used `wp_schedule_single_event` for the Workspace creation job rather than parent's `kdc_qtap_jobs` table because (a) the job is per-user, not per-batch — natural status home is user_meta; (b) parent's job system is shaped for import/export with a results array, awkward fit for an API call.
- Hand-rolled JWT signing avoids a Composer dep on `google/apiclient`. ~30 lines using `openssl_sign(OPENSSL_ALGO_SHA256)`. Token cached in WP transients keyed by (client_email, sub, scope) until expiry minus 60s.
- Service account requires domain-wide delegation in Workspace Admin → API Controls → Domain-wide delegation, with scope `https://www.googleapis.com/auth/admin.directory.user`. Setup guide in the settings page walks through this.
- The `KDC_qTap_Education_Secret` helper is a candidate for promotion to parent if a second qTap plugin needs encrypted option storage.

## [1.0.26] — 2026-04-25

### Added
- **Full-page transaction loader on Create Student submit.** Wired the parent qTap App's `KdcQtapUI.showPageLoader()` (introduced in `kdc-qtap` v2.7.6+, see `kdc-qtap/docs/CHILD-PLUGIN_PAGE-LOADER.md`) around the only server-mutating transaction in this plugin: the Create Student form POST. While the request is in flight the user sees a centered overlay with "Creating student…", switching to "Finalizing…" once the server returns and the success card is rendering. Hidden in `.finally()` so the overlay can never get stuck. Routine fetches (search, slabs refetch, username availability check, associated-user autocomplete) intentionally do NOT use the page loader — they have inline status indicators per the parent's "When NOT to use it" guidance.
- **Explicit `kdc-qtap-frontend-helpers` dependency on the viewScript.** Pre-registered the dashboard block's viewScript handle with the helpers script as a hard dependency so `window.KdcQtapUI` is guaranteed to be defined before `view.js` runs, regardless of theme enqueue order.

## [1.0.25] — 2026-04-25

### Changed
- **Use parent's `--kdc-qtap-radius-md` directly instead of a custom variable.** v1.0.23 introduced `--qtap-edu-field-radius` to unify field/button radii — but the parent already exposes `--kdc-qtap-radius-md` (the same token its own `kdc-qtap-btn` uses). Dropped the custom var, dropped the `.kdc-qtap-btn` override (parent already sets it correctly), and pointed all 12 field-shaped elements at `var(--kdc-qtap-radius-md)`. One token, parent-controlled, consistent across all qTap dashboards.

## [1.0.24] — 2026-04-25

### Changed
- **Use parent qTap color tokens.** Swept 49 hardcoded color literals (`#2271b1`, `#d63638`, `#1f7a1f`, `#646970`, `#1d2327`, `#d4a017`, `#5d4500`, `#5d2226`) and replaced them with the parent's CSS variables (`--kdc-qtap-color-primary`, `--kdc-qtap-color-error`, `--kdc-qtap-color-success-dark`, `--kdc-qtap-color-text-muted`, `--kdc-qtap-color-input-text`, `--kdc-qtap-color-warning`, `--kdc-qtap-color-warning-dark`, `--kdc-qtap-color-error-dark`). Any future qTap brand-color change in `kdc-qtap` now flows through to this dashboard automatically. Same pattern should be adopted by all qTap-series dashboards (Finance, Mobile, etc.) for visual consistency across the suite.

## [1.0.23] — 2026-04-25

### Changed
- **Unified field + button border-radius.** Inputs/selects were `0.25em`, adjustment-row inputs `0.35em`, contact-add `0.3em`, adjustment-add `0.5em`, parent `kdc-qtap-btn` was a fixed 4px — five different radii on visually-similar elements made the form read as patchwork. Introduced a single CSS variable `--qtap-edu-field-radius: 0.4em` on the dashboard wrapper and pointed every field-shaped element (form inputs, search row, contact row, adjustment row, conflict input, control buttons, contact-add, adjustment-add, contact-remove, adjustment-remove, all `.kdc-qtap-btn` instances) at it. Card-shaped elements (slab cards, pill toggles, notice cards, success card, conflict alert) keep their larger radii intentionally — that's the field/card distinction.

## [1.0.22] — 2026-04-25

### Added
- **Live username availability check.** New `GET /students/check-username?username=...` REST endpoint. The Create form fires it 350ms after the staff stops typing first/last; the preview shows a `✓ Available` / `⚠ Taken` badge next to the username so conflicts are caught before submit, not after.
- **Editable conflict-resolution field.** When the live check finds a conflict, an inline editable input appears inside the preview card with the suggested suffixed alternative (`zuvi.kudmule1`) pre-filled. Staff can pick a different username and click "Check" to re-verify. On submit, `username_override` is sent and the server uses it verbatim (no auto-suffix). When there's no conflict the alert stays hidden and submit goes through unchanged.
- **Server respects `username_override`.** `POST /students` validates the override against the Workspace-safe regex + a fresh `username_exists()` check; returns 409 if the override was claimed between the live check and submit.

### Changed
- **Search row matches Create form field size for real this time.** v1.0.21's override only set `min-height` while Finance's staff-find ships fixed `height: 34px` and `font-size: 13px`. Now the dashboard scope explicitly resets `height: auto` plus matches the create-form padding (`0.45em 0.6em`) and border so the search input/select line up exactly with First Name / Last Name / Gender.
- **Preview labels.** `Username (Google Workspace)` → `Username` (the qualifier was redundant); `Default email` → `Default email (Google Workspace)` (the email IS the Workspace login, so the qualifier moves there).

## [1.0.21] — 2026-04-25

### Changed
- **Search row inputs match the Create form's field size.** Finance's `staff-find__field` + `__input` ship their own (smaller) padding/min-height that didn't line up with our larger Create form on this theme. Added a dashboard-scoped override (1em font / 0.7em 0.9em padding / 2.75em min-height) so the search row's "All fields" select and "Type a name…" input match First Name / Last Name / Gender. Override is scoped to `.qtap-education-dashboard` so Finance's own pages aren't affected.
- **Dashboard's `.kdc-qtap-btn` overrides em-based.** The parent ships fixed-pixel font sizes via CSS variables, which on this theme resolved smaller than the surrounding 1em form labels. Now every `.kdc-qtap-btn` instance inside the dashboard reads at 1em / 2.5em min-height (`--lg`: 2.75em / 0.7-1.25em padding) so all action buttons line up with the form text.

## [1.0.20] — 2026-04-25

### Changed
- **Unified button scale across the dashboard.** The form's Create + Cancel pair was small relative to inputs, the top-bar Find + Create-new-{Student} pair didn't match it, and the dashed "+ Add" buttons (contacts, adjustments, expand/collapse) sat at 0.85em while everything around them was 1em. Now:
  - All `kdc-qtap-btn` instances on the dashboard use the `--lg` modifier (44px min-height, 12/24px padding, larger font) for a single primary scale.
  - Icons in those buttons bumped to 18px to keep the proportion.
  - Dashed action buttons (`__contact-add`, `__adj-add`, `__control-btn`) bumped to 1em / 500-weight and 2.75em min-height so they align with `--lg`.
- Net effect: every action button on the create flow reads at the same visual weight, no more "small in some places, big in others."

## [1.0.19] — 2026-04-25

### Fixed
- **Resting × button contrast.** v1.0.17's hover fix made the hover state look great but the resting state's `opacity: 0.6` + neutral border made the × icon look nearly invisible by comparison. Switched the resting state to a visible red glyph (`color: #d63638`) with a 35%-alpha red border, so the affordance reads as destructive at rest too. Hover behavior unchanged (solid red bg, white glyph). Same fix applied to the adjustment-row × button.

## [1.0.18] — 2026-04-25

### Changed
- **Find + Create new {Student} now match.** They're peer call-to-actions in the same control bar, so both use `kdc-qtap-btn--primary`. Previously Create was `--secondary` (transparent/outline) which read as a different visual weight on the host theme. The form's submit/cancel pair keeps its primary/secondary hierarchy since that's the right pattern *inside* a form.

## [1.0.17] — 2026-04-25

### Changed
- **Contacts repeater is capped at qTap Mobile's per-user max.** Reads the parent option `kdc_qtap_block_default_max_numbers` (1-20, default 5) and disables the "Add another contact" button at the limit. Add button now also shows a `(n / max)` counter so the staff sees where they are. REST endpoint `POST /students` defensively trims any oversized payload before save.
- **Hover contrast bumped on destructive buttons.** The contact-row × and adjustment-row × hover states were a faint pink tint that read as nearly disabled; now solid `#d63638` with white glyph + matching focus state. The "Add another contact" hover also flips to solid blue with white text instead of a tinted background.

## [1.0.16] — 2026-04-25

### Added
- **Live fee-slab amounts + due dates in the Create form.** New `GET /kdc/v1/qtap/education/slabs?year&grade` REST endpoint wraps `KDC_qTap_Finance_Enrollment::get_available_slabs_for_enrollment()` and returns each applicable slab with its amount, due date, and exempt flag. The Create form's slab cards refetch when the year/grade selectors change and now show amount + `Due: <date>` per card. Slabs that don't apply to the chosen grade are hidden and unticked so they aren't submitted.
- **Server-side currency formatting.** Slab amounts are formatted via `kdc_qtap_finance_format_amount()` so currency rendering matches Finance throughout the site (symbol, decimals, locale grouping). The endpoint returns both `amount` (raw) and `amount_formatted` (display string).

### Changed
- **Contact mobiles are normalized to E.164 on save.** If the input already starts with `+` it's stored as full E.164. Otherwise the dial code from the parent's `kdc_qtap_settings[phone][default_country]` is prepended (e.g. `9876543210` → `+919876543210` for `IN`). New private helpers `to_e164()` + `get_default_dial_code()` in the REST controller.
- **Tightened the unified font scale + bumped base size.** The form still felt small on the host theme even after v1.0.15's collapse to 5 sizes; bumped the wrapper to `font-size: 1.125em` so all em-based children scale up uniformly. Also collapsed remaining outliers (1.2 / 1.1 / 1.05 / 0.95 / 0.9em → canonical 1 or 0.85em). The 5-step scale is now the only set in use: `2 / 1.25 / 1 / 0.85 / 0.75em`.

## [1.0.15] — 2026-04-25

### Changed
- **Unified the form's font scale.** The dashboard had grown 8 distinct micro-sizes (0.7 / 0.75 / 0.8 / 0.85 / 0.9 / 0.95 / 1 / 1.5 / 2.5em) which read as visual noise. Collapsed to a 5-step scale used consistently:
  - `2.5em` — hero count number (kept)
  - `1.5em` — dashboard title (kept)
  - `1em` — body: inputs, buttons, primary text, names, descriptions, contact-row inputs, adjustment-row inputs/notes, search empty/status, form status
  - `0.85em` — small: field labels, fieldset legends, helper text, slab descriptions, contact-add / adjustment-add buttons, pill-toggle `--sm`
  - `0.75em` — micro: badge, chevron
- Affected ~12 selectors. Result: form chrome reads as one consistent surface instead of N micro-zoomed regions.

## [1.0.14] — 2026-04-25

### Fixed
- **Slab card text was washed out.** The `.__field span { opacity: 0.7 }` rule cascaded to nested spans inside the slab cards (`.__slab-card__name` etc.), muting the text. Tightened the selector to `.__field > span` (direct child only) so descendant spans keep their own colors. Replaced `opacity` with explicit `color` so semibold-on-light reads cleanly. Slab description colour also bumped slightly darker.

### Changed
- **Google + Send-login + Exempt toggles now use the slab-card visual.** Previous render was a small rounded pill; new render is a rectangular card with a 1.4em check + label, matching `Fee Categories`. Single class change (the base `.__pill-toggle` rules now produce slab-card geometry); the existing `--sm` modifier still produces the small pill used for the Adjustments multi-select chips.

## [1.0.13] — 2026-04-25

### Fixed
- **Search result year was always blank.** `KDC_qTap_Finance_Enrollment::get_current()` returns the enrollment payload but the academic year is the *outer* key in Finance's `kdc_qtap_finance_enrollments` user_meta, not a field inside it. The search endpoint now resolves the year separately via `kdc_qtap_finance_get_current_academic_year()` so the meta line shows `Grade · Division · [Year]` instead of `Grade · Division · []`.

### Changed
- **Adjustments section starts empty.** The default empty row was confusing — looked like data the staff needed to fill. Now the first row appears only when `+ Add Discount / Surcharge` is clicked. Implemented via `<template data-role="adj-template">` so the template content isn't even in the rendered DOM until JS clones it.
- **Required fields** (HTML + REST validation): `Gender`, `Academic Year`, `Grade`. Form uses `required` attributes for native browser feedback; client-side `submitForm()` guards before fetch; REST endpoint returns `400 qtap_edu_missing_gender` / `qtap_edu_missing_enrollment` if either is empty.
- **Mobile + Email inputs upgraded** with `inputmode` + `autocomplete` so mobile keyboards open in the right layout (numeric keypad for tel, email keyboard for email). Mobile also has a `pattern="[0-9+\-\s()]*"` so non-numeric chars get a soft warning on submit.
- **Exempt from fees + Adjustments' "Apply to" slab checkboxes** restyled as pill-toggles to match the Google Workspace toggles + slab cards. No more bare browser checkboxes anywhere in the form. Adjustments uses a `--sm` variant (tighter padding, smaller font) so the chips read as multi-select tags rather than full toggles.

## [1.0.12] — 2026-04-25

### Added
- **Custom search REST endpoint** `GET /kdc/v1/qtap/education/students/search?q=…&field=…&roles=…` that joins the user record with the current-year Finance enrollment in one round-trip. Returns `{users: [{id, name, first_name, last_name, email, login, year, grade, division, edit_url}]}`. Replaces the core `/wp/v2/users` call so each search result card can show enrollment context without an N+1 fetch from the client.
- **Search result card meta line** — `Grade · Division · [Year]` (only the bits that are populated). Mirrors Finance staff-console's `__meta` line shape.
- **Live debounced search restored** (350 ms after the last keystroke) on top of the explicit Find button. Switching the field selector with a query already in the box re-fires the search.
- **Adjustments section** in the Enrollment fieldset — repeatable rows for Discounts / Surcharges. Each row: type (Discount / Surcharge), label, mode (Fixed / %), amount, optional notes, and optional per-slab targeting checkboxes (leave all unchecked → applies to every slab). The whole `adjustments[]` array passes through to `KDC_qTap_Finance_Enrollment::save()`, which is Finance's canonical handler for these values, so no parallel logic.
- **`Exempt` badge on Fee Slab cards** when the slab is marked exempt in `kdc_qtap_finance_settings.fees_slabs.{slug}.exempt`.

### Changed
- **Fee slabs are checked by default** — most enrollments use the full slab set, so default-on saves clicks. Staff still uncheck any that don't apply.
- **Send login to Contacts auto-disables when Create Google Account is off.** No Workspace account = nothing to send. Toggle uncheckes itself + becomes greyed-out + non-clickable until the staff re-enables Create Google Account.
- **Payment Cycle placeholder**: `— auto —` → `-- Default --` to match the user's preferred phrasing.

## [1.0.11] — 2026-04-25

### Added
- **Search results + Associated User picker** now render with Finance's `.kdc-qtap-finance-student-card` markup (responsive grid, name + email, hover lift) so they look identical to the Finance staff-console search.
- **Payment Cycle** field in the Enrollment fieldset (Monthly / Quarterly / Half-Yearly / Full Year / Full Cycle / Full Tenure — sourced from `KDC_qTap_Finance_Fee_Matrix::COLLECTION_MODES`). Empty value falls through to Finance's auto-resolution.
- **Two Google Workspace toggles** under the username/email preview:
  - **Create Google Account** *(default ON)* — controls whether the REST response surfaces the seed password. Off = local-only WP user, no password returned.
  - **Send login to Contacts** *(default OFF)* — when on, after creating the user the REST endpoint dispatches the credentials to each contact's email/WhatsApp via the parent's `kdc_qtap_send_notification()` framework.
- **Slab cards (Fee Categories)** — replaced the row of bare browser checkboxes with a responsive grid of pill-cards. Hidden checkbox + state-aware `__check` indicator; descriptions surface inline if Finance settings provide one.
- **Server-side validation: at least one contact required** (with name, mobile, or email). Mirrored client-side in `submitForm()` so the staff sees the error before hitting the network.

### Changed
- **Custom checkboxes throughout the form.** Browser-default checkboxes looked unfit beside Finance's UI; replaced with a hidden native input + painted `__check` indicator (Lucide check on tick). Affects the Exempt-from-fees toggle, the two Google Workspace toggles, and the Fee Categories cards.
- **Bigger border-radii across the form** for a softer, more current look — buttons, inputs, selects, fieldsets, success cards, slab cards all bumped to `0.5em` / `0.6em` / `0.7em` from `0.25em` / `0.3em`.
- **Button classes use parent's `.kdc-qtap-btn` directly** instead of `kdc_qtap_get_button_class()`. The theme-detecting helper was returning a bare `.button` class on Tridha that inherited zero color from the active Beaver Builder theme. Parent's qTap button family is unconditional and CSS-loaded via `kdc_qtap_enqueue_frontend_components()`, so styling is now guaranteed regardless of theme.
- **Associated User label**: "(parent / guardian)" → "(Sibling)" — matches the more common case for new student creation (parent connection is via the `_parent` email suffix; the explicit picker is for sibling / family linkage).
- **Live preview area** now also houses the two Google Workspace toggles, framed with a dashed separator so the relationship between the generated username/email and the Workspace intent is visible at a glance.

### Fixed
- v1.0.10 follow-up: search results container now reliably visible because `setStatus()` toggles `.is-info` / `.is-success` / `.is-error` (Finance's CSS hides the bare container).

## [1.0.10] — 2026-04-25

### Fixed
- **Search results were rendering into a hidden container.** Finance's `.kdc-qtap-finance-staff-find-result` is `display: none` by default and only becomes visible when one of `.is-success` / `.is-info` / `.is-error` is also set on it. v1.0.9 set `innerHTML` but not the state class, so clicking **Find** appeared to do nothing — the markup was correct, the result was just hidden under the cushion. Now `setStatus( html, state )` toggles the state class together with content and clears it on empty queries. Also added a small CSS override to suppress Finance's coloured tint when the container holds our search list (the green/blue background would clash with the rendered list of users).

## [1.0.9] — 2026-04-25

### Changed
- **Adopted the qTap-series UI conventions across the dashboard** so the Education block looks like the Finance staff console rather than a one-off. The render callback now enqueues `kdc_qtap_enqueue_frontend_components()` (parent's button/input framework) plus Finance's `kdc-qtap-finance-staff-console` stylesheet, and the markup uses the same primitives:
  - Top-level **Count** / **{Student}** tabs render as `.kdc-qtap-finance-staff-menu` with Lucide icons (`bar-chart-3`, `user`) via the parent's `kdc_qtap_lucide()` helper. Active state is mapped from our hidden radio's `:checked` to the same visual treatment Finance uses for `.is-active`.
  - Header uses Finance's `.kdc-qtap-finance-staff-dashboard__header` pattern (title + tagline left).
  - **{Student} → Find** is now the Finance `.kdc-qtap-finance-staff-find` pattern: field selector (All fields / Name / Email) + input + explicit **Find** button. Search is **click-to-find** (matches Finance's Receipts UX), no longer live-debounced.
  - Form Submit / Cancel / Find / "+ Create new {Student}" buttons all resolve through `kdc_qtap_get_button_class('primary' | 'secondary')` so they pick up the active theme's button style (WooCommerce, Block theme, or qTap custom UI mode).

### Added
- **Username = `first.last`** (lowercased, Workspace-sanitized — diacritics stripped, special chars removed; first char forced to a letter; numeric suffix only on collision, with a server-side warning).
- **Email = `<username>_parent@<site_host>`** (e.g. `zuvi.kudmule_parent@tridha.edu.in`). The `_parent` suffix flags the mailbox as the parent/guardian contact since students don't have their own email yet.
- **Random 20-char password** generated and **returned in the REST response** so staff can copy it once into Google Workspace setup. Site auth itself uses Google Login or WhatsApp OTP — the password is purely a Workspace seed and is never re-surfaced.
- **Live username + email + warnings preview** under the name fields. As the staff types First / Last name, the UI shows the exact `first.last` and `first.last_parent@…` strings that the server would generate, plus inline warnings if the input contains spaces or characters Google Workspace will reject. Mirrors the server-side sanitization rules (NFD-strip diacritics + lowercase + `[^a-z0-9.\-]` strip).
- **Success card** after create: shows the new user's name + edit-link, the assigned login + email, the generated password with a one-click **Copy** button, and any server-side warnings (e.g. "first.last was already taken; first.last2 was assigned instead").

## [1.0.8] — 2026-04-25

### Added
- **Two-level tabs.** Top-level **Count** | **{Student}** sections. The whole previous block (Grade / Division / Exempt / Unassigned) now lives inside Count. Pure-CSS toggling (radio + `:checked`) at both levels — distinct class prefixes (`__section-*` outer, `__tab-*` inner) so the two layers don't fight each other.
- **{Student} tab — Search.** Live debounced search field (300 ms) calling `/wp/v2/users?search=…&roles=kdc_qtap_student,kdc_qtap_parent,subscriber&context=edit`. Each result links to `wp-admin/user-edit.php?user_id=…` in a new tab. Min query length 2 chars; results show last/first name, email, roles, and the user ID. Uses the `wp_rest` nonce so any user with a REST-API role configured in qTap App Settings can search.
- **{Student} tab — Inline create form.** Toggleable (no page reload). Captures First Name, Last Name, Gender (from Finance `gender_options`), Date of Birth, an Associated User (parent/guardian, picked via the same kind of debounced search), a **repeatable Contact list** (Name / Mobile / Email — kdc-qtap-mobile's `kdc_qtap_mobile_numbers` shape), and an **Enrollment** block (Academic Year, Grade, Division, Fee Slabs multi-select, Exempt flag) — all sourced from Finance settings. Submit goes to a new REST endpoint and reports success/failure inline; the "+ Create new" button toggles it open/closed without nuking page state.
- **REST endpoint** `POST /wp-json/kdc/v1/qtap/education/students`, gated by `kdc_qtap_rest_permission_check()` (parent's REST role config). Creates a WP user with role `kdc_qtap_student`, sets `first_name`, `last_name`, `display_name`, `kdc_qtap_finance_gender`, `kdc_qtap_finance_dob`, `kdc_qtap_mobile_numbers`, runs Finance's `sync_associated_users()` for bidirectional parent/student linking when an associated user is provided, and calls `KDC_qTap_Finance_Enrollment::save()` so the enrollment row is written via Finance's canonical helper (which auto-assigns applicable fee slabs when the form leaves the slab boxes unchecked). Returns `{user_id, login, name, edit_url, enrolled}`.
- New class `KDC_qTap_Education_REST_API` wired through `load_dependencies()` + `init_components()`. Single endpoint today, room to grow into a `qtap-education` Abilities API surface later — same auth path the planned parent mobile app will use.

## [1.0.7] — 2026-04-25

### Changed
- **Replaced the Unicode chevron `▶` with an inline Lucide `chevron-right` SVG** to comply with the project's frontend icon policy (Lucide on frontend, Dashicons in WP admin, no emojis, no `$` / currency glyphs). The SVG inherits `currentColor` and uses the existing `transform: rotate(90deg)` for the expand state, so the visual behaviour is unchanged.
- **Removed the middle-dot (`·`) marker on student rows** in the Unassigned tab; it was decorative only and not a Lucide icon. The existing column indent already conveys the hierarchy. Empty leading cell preserves alignment.

## [1.0.6] — 2026-04-25

### Added
- **4th tab: "Unassigned"** — per-grade list of students whose enrollment has no division. Each row shows last name, first name, and the user_id rendered as a link to `wp-admin/user-edit.php?user_id=X` opened in a new tab. No gender breakdown — this is an operational list, not an aggregate. Uses the same expand/collapse + Expand-all/Collapse-all controls as the other tabs.
- **Finance custom labels respected throughout the dashboard.** Tab labels, column headers, summary band, and empty-state copy now resolve through `KDC_qTap_Education_Dashboard::label()` → `kdc_qtap_finance_label()` so an institution that calls grades "Levels" and divisions "Sections" sees those words automatically. Falls back to English defaults if Finance's helper isn't loaded.
- `KDC_qTap_Education_Dashboard::build_unassigned_view()` data builder + `fetch_user_names()` batched query (one SQL on `wp_users` for `display_name`, one on `wp_usermeta` for `first_name` / `last_name`, no N+1).
- `KDC_qTap_Education_Dashboard::label( $key, $plural )` public helper for any future code that needs Finance-aware labels.

### Changed
- **Renamed the "RTE" tab → "Exempt"** because Finance's `exempt` flag is generic — it could mark RTE seats, scholarship students, staff-family discounts, or anything else exempted from fees. "RTE" was inaccurately specific. Internal view key also renamed from `'rte'` → `'exempt'`. CSS selectors updated.
- View-summary shape: each view now carries a `'type'` field (`'aggregate'` for Grade/Division/Exempt, `'list'` for Unassigned). render.php dispatches to the right panel renderer based on it.

## [1.0.5] — 2026-04-25

### Added
- **Expand-all / Collapse-all controls + per-row toggle** for each tab's table. Outer rows (grades in Grade/RTE tabs, divisions in Division tab) are clickable and now show a small chevron that rotates 90° when expanded. Sub-rows are server-rendered with `is-hidden` so the **default state on first paint is collapsed** — no flash of expanded content before the JS attaches. Keyboard-accessible (Enter / Space toggle, Tab focuses outer rows, role=button + aria-expanded).
- New `blocks/dashboard/view.js` (~80 lines, vanilla, no dependencies) registered as the block's `viewScript` so WordPress only loads it on pages that contain the block.

### Changed
- Outer rows now have hover styling and `cursor: pointer` to invite the click. `user-select: none` so dragging-to-select doesn't fight the toggle.

### Added
- **Three tabs on the dashboard**: **Grade** (default — same view as before), **Division** (inverse grouping — outer = division, sub-rows = grades), and **RTE** (Grade > Division but only counting enrollments where Finance's `exempt` flag is `true`). Each tab gets its own total + per-gender pills + breakdown table; the RTE tab's total reflects only exempt students. Tab switching is pure CSS (radio + `:checked ~ .panel--X`) — no JavaScript runtime, no AJAX, accessible via keyboard, multiple instances of the block on the same page get unique radio names so they don't interfere.
- `KDC_qTap_Education_Dashboard::build_view()` private helper that builds any outer-by-inner view from one enrollment iteration. Accepts an optional filter callable so the same code path serves the Grade, Division, and RTE views (the RTE view passes `fn($e) => !empty($e['exempt'])`).

### Changed
- Output shape of `KDC_qTap_Education_Dashboard::build_summary()` changed: dropped the top-level `total`, `by_gender`, `grades` keys in favour of a single `views` map (`grade`, `division`, `rte`), each with its own `total`, `by_gender`, and `rows` (outer → sub). The `gender_keys` and `year` keys at the top level are unchanged. **Breaking** for any external code that read the old shape — but the dashboard block was the only consumer.

## [1.0.3] — 2026-04-25

### Changed
- **Hide gender columns whose total is 0** across all enrollments. Previously the table always rendered every configured `gender_options` value plus an `Unspecified` bucket, leaving entire columns of zeros (e.g. `Other`, `Unspecified` on Tridha). Now those columns drop out of both the summary pills and the table — the column header is computed once from `summary['by_gender']` totals and pruned before rendering.
- **All font sizes (and proportional spacing) switched from `rem` to `em`.** The dashboard root no longer sets `font-size`, so it inherits whatever size the theme/wrapper provides; every child uses `em` so the whole component scales together. To make the dashboard larger or smaller you can now set `font-size` on a wrapping element — no plugin update needed. Reasonable defaults: title `1.5em`, total value `2.5em`, total label / table headers `0.75em`, pills / table body `1em`. Padding, gaps, border-radii also converted so they remain proportional at any size.
- **Pill colour helpers now also match `--boy` / `--girl`** in addition to `--male` / `--female`, since Tridha (and likely other Indian schools) configures their `gender_options` as Boy/Girl. The colour cue now carries through regardless of label phrasing.

## [1.0.2] — 2026-04-25

### Changed
- **Doubled font sizes** across the dashboard for projection/screen-share readability — title 1.75→3.5rem, meta 1→2rem, total value 2.25→4.5rem, total label 0.85→1.7rem, summary pills 1→2rem (with proportional padding), table 1.05→2.1rem (with proportional cell padding), header row 0.875→1.75rem.

## [1.0.1] — 2026-04-25

### Changed
- **Dashboard rows now follow Finance grade order** instead of alphabetical. Previously `I, II, III, IV, IX, Jr. K.g.` (alphabetical via `ksort`); now `Playgroup, Nursery, Jr. K.g., Sr. K.g., I, II, …, X, AS & A` matching the **Finance > Settings > Grades/Classes** configured order. Same fix applied to divisions (uses `Finance > Settings > Divisions` order); the `Unassigned` bucket always sinks to the end of its grade. Grades or divisions present in enrollment data but missing from Finance settings (e.g. legacy data) fall through alphabetically after the configured ones, so they remain visible.
- **Bumped font sizes** across the dashboard for better readability — title 1.5→1.75rem, total 1.85→2.25rem, summary pills 0.9→1rem, table 0.95→1.05rem with larger cell padding, header row 0.8→0.875rem. Per-cell gender count is now slightly less muted (rgba 0.7→0.85) so zero values still read clearly.

## [1.0.0] — 2026-04-25

### Added
- Initial plugin scaffold.
- Dependency checks for `kdc-qtap` (parent) and `kdc-qtap-finance`.
- Registration with the qTap App dashboard at priority 30.
- Admin submenu placeholder under qTap App ("Education").
- Helper functions: `kdc_qtap_education_is_active()`, `kdc_qtap_education_version()`, `kdc_qtap_education_get_settings()`.
- **Frontend dashboard block** `qtap/education-dashboard`. Server-rendered, gated by `is_user_logged_in()` + `kdc_qtap_can_access_rest_api()` (REST role list managed in qTap App > Settings). Shows total enrollments for the current academic year and a `Year > Grade > Division` breakdown table with per-row gender counts. Gender values come from Finance's `kdc_qtap_finance_gender` user_meta and `gender_options` setting; users with no recorded gender fall into an "Unspecified" bucket. Single batched usermeta query — no AJAX, no caching.
- `KDC_qTap_Education_Dashboard::build_summary( $year = '' )` data builder (defaults to current academic year). Returns total + per-gender + per-grade + per-division counts as a flat array consumable by the block template or future REST/Abilities endpoints.

### Reserved (planned)
- `includes/abilities/` — WP Abilities API surface for the parent mobile app and other connectors. Will live inside this plugin (not a sibling) so the plugin remains self-contained. Ability category: `qtap-education`.
