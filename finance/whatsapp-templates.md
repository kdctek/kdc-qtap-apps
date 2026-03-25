# qTap Finance — WhatsApp Utility Message Templates

Reference for registering WhatsApp Business API **Utility** templates in Meta Business Manager.
Each template maps to a notification type in the plugin.

---

## Notification Types

| # | Internal Key | Display Name | Default | Audience |
|---|-------------|--------------|---------|----------|
| 1 | `payment_due_reminder` | Payment Due Reminder | Enabled | User |
| 2 | `payment_overdue` | Payment Overdue | Enabled | User |
| 3 | `payment_received` | Payment Received | Enabled | User |
| 4 | `payment_refunded` | Payment Refunded | Enabled | User |
| 5 | `payment_verified` | Offline Payment Verified | Enabled | User |
| 6 | `payment_rejected` | Offline Payment Rejected | Enabled | User |
| 7 | `offline_submitted` | Offline Payment Submitted | Enabled | Admin |
| 8 | `enrollment_created` | Enrollment Created | Disabled | User |
| 9 | `fee_assigned` | Fee Assigned | Disabled | User |

> **Plugin prefix:** `finance_` — the full type used internally is e.g. `finance_payment_due_reminder`.

---

## Template Proposals

### 1. Payment Due Reminder

**Suggested template name:** `payment_due_reminder_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | Payment Reminder |
| **Body** | Hi {{1}}, this is a reminder that your {{2}} payment for {{3}} ({{4}}-{{5}}) is due on {{6}}. Outstanding balance: {{7}}. Please pay on time to avoid late fees. |
| **Footer** | {{8}} |
| **Buttons** | URL — `Pay Now` → {{9}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{fee_category}}` | Tuition Fee |
| {{3}} | `{{academic_year}}` | 2025-2026 |
| {{4}} | `{{grade}}` | Grade 5 |
| {{5}} | `{{division}}` | A |
| {{6}} | `{{due_date:short}}` | Apr 5, 2025 |
| {{7}} | `{{balance:lakh}}` | ₹15,000 |
| {{8}} | `{{institute_name}}` | ABC School |
| {{9}} | `{{payment_url}}` | https://example.com/pay/... |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}\n{{due_date}}\n{{balance}}
footer:  {{institute_name}}
buttons: {{payment_url}}
```

---

### 2. Payment Overdue

**Suggested template name:** `payment_overdue_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | Payment Overdue |
| **Body** | Hi {{1}}, your {{2}} payment for {{3}} ({{4}}-{{5}}) is now {{6}} days overdue. Outstanding balance: {{7}}. Please pay immediately to avoid further action. |
| **Footer** | {{8}} |
| **Buttons** | URL — `Pay Now` → {{9}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{fee_category}}` | Tuition Fee |
| {{3}} | `{{academic_year}}` | 2025-2026 |
| {{4}} | `{{grade}}` | Grade 5 |
| {{5}} | `{{division}}` | A |
| {{6}} | `{{days_overdue}}` | 12 |
| {{7}} | `{{balance:lakh}}` | ₹15,000 |
| {{8}} | `{{institute_name}}` | ABC School |
| {{9}} | `{{payment_url}}` | https://example.com/pay/... |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}\n{{days_overdue}}\n{{balance}}
footer:  {{institute_name}}
buttons: {{payment_url}}
```

---

### 3. Payment Received (with Order Details)

**Suggested template name:** `payment_received_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | Payment Confirmation |
| **Body** | Hi {{1}}, we've received your payment of {{2}} for {{3}} ({{4}} - {{5}}-{{6}}). Payment method: {{7}}. Reference: {{8}}. Order #{{9}}. Thank you! |
| **Footer** | {{10}} |
| **Buttons** | URL — `View Receipt` → {{11}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{transaction_amount:lakh}}` | ₹15,000 |
| {{3}} | `{{fee_category}}` | Tuition Fee |
| {{4}} | `{{academic_year}}` | 2025-2026 |
| {{5}} | `{{grade}}` | Grade 5 |
| {{6}} | `{{division}}` | A |
| {{7}} | `{{payment_method}}` | Online - Razorpay |
| {{8}} | `{{reference_number}}` | UTR123456789 |
| {{9}} | `{{wc_order_id}}` | 1042 |
| {{10}} | `{{institute_name}}` | ABC School |
| {{11}} | `{{receipt_url}}` | https://example.com/receipt/... |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{transaction_amount}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}\n{{payment_method}}\n{{reference_number}}\n{{wc_order_id}}
footer:  {{institute_name}}
buttons: {{receipt_url}}
```

> **Note:** `{{wc_order_id}}` is only populated for WooCommerce payments. For offline/direct payments, this slot will be empty — consider a simpler variant without order ID if needed.

---

### 4. Payment Refunded (with Order Details)

**Suggested template name:** `payment_refunded_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | Refund Processed |
| **Body** | Hi {{1}}, a refund of {{2}} has been processed for {{3}} ({{4}} - {{5}}-{{6}}). Order #{{7}}. The refund will be credited to your original payment method. |
| **Footer** | {{8}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{refund_amount:lakh}}` | ₹5,000 |
| {{3}} | `{{fee_category}}` | Tuition Fee |
| {{4}} | `{{academic_year}}` | 2025-2026 |
| {{5}} | `{{grade}}` | Grade 5 |
| {{6}} | `{{division}}` | A |
| {{7}} | `{{wc_order_id}}` | 1042 |
| {{8}} | `{{institute_name}}` | ABC School |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{refund_amount}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}\n{{wc_order_id}}
footer:  {{institute_name}}
buttons:
```

---

### 5. Offline Payment Verified

**Suggested template name:** `payment_verified_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | Payment Verified |
| **Body** | Hi {{1}}, your offline payment of {{2}} for {{3}} ({{4}} - {{5}}-{{6}}) has been verified and approved. Thank you! |
| **Footer** | {{7}} |
| **Buttons** | URL — `View Receipt` → {{8}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{transaction_amount:lakh}}` | ₹15,000 |
| {{3}} | `{{fee_category}}` | Tuition Fee |
| {{4}} | `{{academic_year}}` | 2025-2026 |
| {{5}} | `{{grade}}` | Grade 5 |
| {{6}} | `{{division}}` | A |
| {{7}} | `{{institute_name}}` | ABC School |
| {{8}} | `{{receipt_url}}` | https://example.com/receipt/... |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{transaction_amount}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}
footer:  {{institute_name}}
buttons: {{receipt_url}}
```

---

### 6. Offline Payment Rejected

**Suggested template name:** `payment_rejected_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | Payment Not Verified |
| **Body** | Hi {{1}}, your offline payment of {{2}} for {{3}} ({{4}} - {{5}}-{{6}}) could not be verified. Reason: {{7}}. Please contact us or submit a new payment. |
| **Footer** | {{8}} |
| **Buttons** | URL — `Submit Payment` → {{9}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{transaction_amount:lakh}}` | ₹15,000 |
| {{3}} | `{{fee_category}}` | Tuition Fee |
| {{4}} | `{{academic_year}}` | 2025-2026 |
| {{5}} | `{{grade}}` | Grade 5 |
| {{6}} | `{{division}}` | A |
| {{7}} | `{{rejection_reason}}` | UTR number not found |
| {{8}} | `{{institute_name}}` | ABC School |
| {{9}} | `{{payment_url}}` | https://example.com/pay/... |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{transaction_amount}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}\n{{rejection_reason}}
footer:  {{institute_name}}
buttons: {{payment_url}}
```

---

### 7. Offline Payment Submitted (Admin)

**Suggested template name:** `offline_payment_submitted_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | New Payment Submission |
| **Body** | A new offline payment requires verification. User: {{1}}. Fee: {{2}} ({{3}} - {{4}}-{{5}}). Amount: {{6}}. Method: {{7}}. Reference: {{8}}. |
| **Footer** | {{9}} |
| **Buttons** | URL — `Verify Now` → {{10}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{fee_category}}` | Tuition Fee |
| {{3}} | `{{academic_year}}` | 2025-2026 |
| {{4}} | `{{grade}}` | Grade 5 |
| {{5}} | `{{division}}` | A |
| {{6}} | `{{transaction_amount:lakh}}` | ₹15,000 |
| {{7}} | `{{payment_method}}` | Bank Transfer |
| {{8}} | `{{reference_number}}` | UTR123456789 |
| {{9}} | `{{institute_name}}` | ABC School |
| {{10}} | `{{verify_payments_url}}` | https://example.com/wp-admin/... |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}\n{{transaction_amount}}\n{{payment_method}}\n{{reference_number}}
footer:  {{institute_name}}
buttons: {{verify_payments_url}}
```

> **Recipient:** This notification goes to admin/staff, not the student.

---

### 8. Enrollment Created

**Suggested template name:** `enrollment_created_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | Enrollment Confirmed |
| **Body** | Hi {{1}}, your enrollment has been confirmed. {{2}}: {{3}}. {{4}}: {{5}}. {{6}}: {{7}}. Welcome! |
| **Footer** | {{8}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | Label: Academic Year | Academic Year |
| {{3}} | `{{academic_year}}` | 2025-2026 |
| {{4}} | Label: Grade | Grade |
| {{5}} | `{{grade}}` | Grade 5 |
| {{6}} | Label: Division | Division |
| {{7}} | `{{division}}` | A |
| {{8}} | `{{institute_name}}` | ABC School |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{academic_year}}\n{{grade}}\n{{division}}
footer:  {{institute_name}}
buttons:
```

> **Note:** Label text (Academic Year, Grade, Division) should be hardcoded in the Meta template body since WhatsApp templates don't support dynamic labels. Use the Label Editor defaults matching your business type.

---

### 9. Fee Assigned

**Suggested template name:** `fee_assigned_en`
**Category:** Utility

| Component | Content |
|-----------|---------|
| **Header** | New Fee Assigned |
| **Body** | Hi {{1}}, a fee has been assigned to your account. {{2}} for {{3}} ({{4}}-{{5}}). Amount: {{6}}. Due: {{7}}. |
| **Footer** | {{8}} |
| **Buttons** | URL — `View Details` → {{9}} |

**Variable mapping:**

| Slot | Plugin Variable | Example Value |
|------|----------------|---------------|
| {{1}} | `{{user_name}}` | John Doe |
| {{2}} | `{{fee_category}}` | Tuition Fee |
| {{3}} | `{{academic_year}}` | 2025-2026 |
| {{4}} | `{{grade}}` | Grade 5 |
| {{5}} | `{{division}}` | A |
| {{6}} | `{{amount_due:lakh}}` | ₹15,000 |
| {{7}} | `{{due_date:short}}` | Apr 5, 2025 |
| {{8}} | `{{institute_name}}` | ABC School |
| {{9}} | `{{payment_url}}` | https://example.com/pay/... |

**Plugin config:**
```
header:  {{user_name}}
body:    {{user_name}}\n{{fee_category}}\n{{academic_year}}\n{{grade}}\n{{division}}\n{{amount_due}}\n{{due_date}}
footer:  {{institute_name}}
buttons: {{payment_url}}
```

---

## Available Variables Reference

### User
| Variable | Description |
|----------|-------------|
| `{{user_name}}` | User display name |
| `{{user_email}}` | User email address |
| `{{user_phone}}` | User phone number |
| `{{user_id}}` | WordPress user ID |

### Enrollment
| Variable | Description |
|----------|-------------|
| `{{academic_year}}` | e.g. "2025-2026" |
| `{{grade}}` | Grade/Floor/Category (per Label Editor) |
| `{{division}}` | Division/Wing/Group (per Label Editor) |
| `{{enrollment_status}}` | Enrollment status |

### Payment
| Variable | Description |
|----------|-------------|
| `{{fee_category}}` | Fee slab name |
| `{{amount_due}}` | Total amount due |
| `{{amount_paid}}` | Amount already paid |
| `{{balance}}` | Remaining balance |
| `{{due_date}}` | Payment due date |
| `{{payment_status}}` | Status label |
| `{{payment_id}}` | Payment record ID |
| `{{installment_label}}` | e.g. "Term 1: Apr 2025" |
| `{{installment_amount}}` | Formatted installment amount |
| `{{period_start}}` | Billing period start date |
| `{{period_end}}` | Billing period end date |

### Transaction
| Variable | Description |
|----------|-------------|
| `{{transaction_id}}` | Transaction ID |
| `{{transaction_amount}}` | Transaction amount |
| `{{payment_method}}` | e.g. "Online - Razorpay", "Bank Transfer" |
| `{{payment_date}}` | Transaction date |
| `{{reference_number}}` | UTR/Reference number |
| `{{rejection_reason}}` | Reason for rejection |
| `{{receipt_url}}` | Shareable receipt URL |

### Order Details (WooCommerce)
| Variable | Description |
|----------|-------------|
| `{{wc_order_id}}` | WooCommerce order ID (e.g. 1042) |
| `{{payment_url}}` | Direct payment link |
| `{{receipt_url}}` | Shareable receipt URL |

> **Note:** `{{wc_order_id}}` is only available for WooCommerce-processed payments. For offline/direct payments it will be empty.

### Refund
| Variable | Description |
|----------|-------------|
| `{{refund_amount}}` | Refund amount |
| `{{wc_order_id}}` | Original WooCommerce order ID |

### Institute
| Variable | Description |
|----------|-------------|
| `{{institute_name}}` | Organization name |
| `{{institute_email}}` | Organization email |
| `{{institute_phone}}` | Organization phone |
| `{{institute_address}}` | Organization address |
| `{{currency_symbol}}` | e.g. "₹" |

### System / URLs
| Variable | Description |
|----------|-------------|
| `{{site_name}}` | Website name |
| `{{site_url}}` | Website URL |
| `{{current_date}}` | Current date |
| `{{payment_url}}` | Direct payment link |
| `{{receipt_url}}` | Shareable receipt URL |
| `{{verify_payments_url}}` | Admin verify payments page |
| `{{fees_page_url}}` | Student fees page |

---

## Format Modifiers

Append to any variable with `:modifier` syntax.

### Amount Formats
| Modifier | Output | Example |
|----------|--------|---------|
| `:lakh` | Indian grouping | ₹1,00,000 |
| `:compact` | Short form | ₹1.5L |
| `:words` | In words | One Lakh |
| `:value` | Raw number | 100000 |
| `:international` | Standard grouping | ₹100,000 |

### Date Formats
| Modifier | Output | Example |
|----------|--------|---------|
| `:short` | Short date | Jan 10, 2026 |
| `:long` | Long date | January 10, 2026 |
| `:human` | Relative | 2 days ago |
| `:relative` | Relative | in 3 days |
| `:Y-m-d` | ISO | 2026-01-10 |
| `:d-m-Y` | Indian | 10-01-2026 |

### Text Formats
| Modifier | Output |
|----------|--------|
| `:upper` | UPPERCASE |
| `:lower` | lowercase |
| `:title` | Title Case |

---

## Meta Business Manager Registration

### Steps

1. Go to **Meta Business Suite** → **WhatsApp Manager** → **Message Templates**
2. Click **Create Template**
3. Select **Category: Utility**
4. Set **Language: English**
5. Enter the template name (e.g. `payment_due_reminder_en`)
6. Fill in Header, Body, Footer, and Buttons as shown above
7. Replace numbered placeholders ({{1}}, {{2}}, etc.) with sample values for review
8. Submit for approval (typically approved within minutes for Utility templates)

### Important Notes

- WhatsApp Utility templates use **numbered** placeholders (`{{1}}`, `{{2}}`, etc.)
- The plugin maps its **named** variables to these numbered slots in order
- Body text in the plugin config uses `\n` to separate variables — each line maps to one numbered parameter
- Header and Footer each count as one parameter slot
- Button URLs use the `{{payment_url}}` or `{{receipt_url}}` variable
- Ensure all sample values are realistic when submitting for Meta review
- Templates with URLs in buttons require the URL domain to be verified in Meta Business Manager
