# qTap Finance — A Beginner's Guide

> *Imagine you're the principal of a school. Hundreds of students come every year, and each one has to pay different fees. How do you keep track of who paid what, who still owes money, and who gets a free pass? That's exactly what **qTap Finance** does — it's your digital accountant!*

---

## Table of Contents

1. [What is qTap Finance?](#part-1-what-is-qtap-finance)
2. [The Fee Matrix — Your Price List](#part-2-the-fee-matrix--your-price-list)
3. [Enrollments — Giving Each Student Their Fee Sheet](#part-3-enrollments--giving-each-student-their-fee-sheet)
4. [Recording Payments — The Accountant's Way (Backend)](#part-4-recording-payments--the-accountants-way-backend)
5. [Paying Fees — The Parent's Way (Frontend)](#part-5-paying-fees--the-parents-way-frontend)
6. [The Journey of a Payment](#part-6-the-journey-of-a-payment)

---

## Part 1: What is qTap Finance?

Think of your school like a small business:

| Real World | In qTap Finance |
|-----------|-----------------|
| The school year (April 2025 to March 2026) | **Academic Year** (2025-2026) |
| Classes like "Grade 1", "Grade 5" | **Grades** |
| Sections like "A", "B", "C" | **Divisions** |
| A list of all fees and their prices | **Fee Matrix** |
| Writing a student's name in the register | **Enrollment** |
| The bill you get from a shop | **Payment Record** |
| The receipt after you pay | **Transaction** |

### The 4 Building Blocks

Everything in qTap Finance is built on these four things:

1. **Academic Years** — The school calendar. Example: "2025-2026". You can have multiple years set up at once (this year, next year, etc.)

2. **Grades & Divisions** — How your school organizes students. Grades are the class level (Nursery, LKG, Grade 1...), and Divisions are the sections within each grade (A, B, C...).

3. **Fee Matrix** — The master price list. It tells the system: *"Grade 1 students pay ₹15,000 for Tuition, Grade 5 students pay ₹18,000 for Tuition, everyone pays ₹5,000 for Transport..."*

4. **Enrollments** — The act of saying: *"This student is in Grade 5, Division A, for the year 2025-2026."* The moment you enroll a student, the system automatically creates all their fee records.

---

## Part 2: The Fee Matrix — Your Price List

### What is a "Slab"?

A **slab** is just a fancy word for a **fee category** — a type of fee you charge students.

**Common examples of slabs:**

| Slab Name | What It's For |
|-----------|---------------|
| Tuition Fee | The main school fee for teaching |
| Transport Fee | School bus charges |
| Lab Fee | Science/Computer lab equipment |
| Activity Fee | Sports, music, art activities |
| Exam Fee | Conducting exams |

### How Amounts Differ by Grade

Here's the cool part — the **same slab** can have **different prices** for different grades. Just like a movie ticket costs less for kids and more for adults!

**Example Fee Matrix for 2025-2026:**

| Slab | Nursery | Grade 1 | Grade 5 | Grade 10 | Due Date |
|------|---------|---------|---------|----------|----------|
| Tuition Fee | ₹12,000 | ₹15,000 | ₹18,000 | ₹25,000 | Jun 15 |
| Transport Fee | ₹5,000 | ₹5,000 | ₹5,000 | ₹5,000 | Jun 15 |
| Lab Fee | — | — | ₹3,000 | ₹5,000 | Jul 1 |
| Activity Fee | ₹2,000 | ₹2,000 | ₹2,000 | ₹2,000 | Jun 15 |

Notice how:
- Tuition increases as the grade goes up
- Transport is the same for everyone
- Lab Fee only applies to Grade 5 and above (Nursery and Grade 1 have a "—" meaning it doesn't apply to them)

### What About Free Passes? (Exempt Fees)

Some students get **exemptions** — they don't have to pay certain fees. This could be because of:
- **RTE** (Right to Education) — a government rule that gives free education to some students
- **Scholarships** — the school decides to waive the fee

When a slab is marked as "Can be Exempted," the admin can give specific students a free pass for that fee. Their payment record will show **"Exempt"** instead of a pending amount.

### Step-by-Step: How an Admin Sets Up the Fee Matrix

**Step 1: Set up the basics** (General Tab)

1. Go to **qTap Education** in the admin menu
2. Click the **General** tab
3. Add your **Academic Years** (e.g., "2025-2026", "2026-2027")
4. Add your **Grades** (e.g., Nursery, LKG, UKG, Grade 1, Grade 2...)
5. Add your **Divisions** (e.g., A, B, C, D)
6. Add your **Fee Categories** — these become your slabs (e.g., Tuition Fee, Transport Fee)
7. Click **Save**

**Step 2: Set the current year** (Institute Tab)

1. Click the **Institute** tab
2. Select the **Current Academic Year** from the dropdown
3. Fill in your institute name, email, phone, and currency settings
4. Click **Save**

**Step 3: Build the fee matrix** (Fee Matrix Tab)

1. Click the **Fee Matrix** tab
2. Select the academic year from the dropdown at the top
3. For each fee slab (Tuition, Transport, etc.):
   - Enter the **amount** for each grade
   - Set the **due date** (when should this fee be paid by?)
   - Check **"Can be Exempted"** if some students might get a free pass
   - Uncheck a grade if that slab doesn't apply to it
4. Click **Save Fee Matrix**

**Pro Tip:** If you already set up last year's fees and this year's are similar, use the **"Copy from Year"** button to copy everything and then just adjust the amounts!

---

## Part 3: Enrollments — Giving Each Student Their Fee Sheet

### What Happens When You Enroll a Student?

Enrolling a student is like writing their name in the attendance register — but smarter. Here's what the system does automatically:

1. You tell it: *"Riya is in Grade 5, Division A, for 2025-2026"*
2. The system looks at the Fee Matrix and finds all the fees that apply to Grade 5
3. It creates a **payment record** for each applicable fee:
   - Tuition Fee: ₹18,000 — Due: Jun 15 — Status: **Pending**
   - Transport Fee: ₹5,000 — Due: Jun 15 — Status: **Pending**
   - Lab Fee: ₹3,000 — Due: Jul 1 — Status: **Pending**
   - Activity Fee: ₹2,000 — Due: Jun 15 — Status: **Pending**

That's 4 payment records created automatically! The admin didn't have to manually enter each one.

### What If a Student Gets a Free Pass?

If Riya is an **RTE student** (exempt):
- The system still creates all 4 payment records
- But each one shows: Amount Due = **₹0**, Status = **Exempt**
- Nobody needs to pay anything — the records exist just for bookkeeping

### Step-by-Step: How to Enroll a Student

1. Go to **qTap Education** > **Enrollments** tab
2. You'll see a list of students (or an empty list if you're just starting)
3. To enroll a new student:
   - Select the **Academic Year** (e.g., 2025-2026)
   - Search for the student by name or email
   - Assign their **Grade** (e.g., Grade 5)
   - Assign their **Division** (e.g., A)
   - The system shows which **Fee Slabs** will apply — you can customize this if needed
   - If the student qualifies, check **"Exempt"** for a free pass
4. Click **Save Enrollment**
5. The system instantly creates all the payment records!

### What If You Change the Fee Matrix Later?

No worries! If the school updates the Fee Matrix (say, Transport Fee goes from ₹5,000 to ₹6,000), the system **automatically syncs** the changes:
- Existing payment records get updated due dates
- New slabs that were added get new payment records created
- Nothing gets lost or forgotten

---

## Part 4: Recording Payments — The Accountant's Way (Backend)

> *If you ever want to become an accountant, this section is for you! This is how the school's admin (the person managing fees) handles money from behind the scenes.*

### The Admin's Dashboard

When the admin logs into WordPress, they go to **qTap Education** and see several tabs. For payments, the most important one is:

**Verify Payments** — Think of this as the admin's inbox. Whenever a parent says *"I paid!"*, it shows up here for the admin to check and approve.

### How the Verification Process Works

**Imagine this scenario:**

1. Riya's dad transfers ₹18,000 to the school's bank account for Tuition Fee
2. He goes to the school website and submits the payment details (UTR number, date, screenshot)
3. The payment shows up in the admin's **Verify Payments** tab

**What the admin sees:**

| Field | Value |
|-------|-------|
| Student | Riya (Grade 5 - A) |
| Fee Category | Tuition Fee |
| Amount | ₹18,000 |
| Payment Method | Bank Transfer |
| UTR/Reference | UTIB12345678 |
| Payment Date | June 10, 2025 |
| Receipt | (attached screenshot) |
| Notes | "Paid via NEFT" |

**The admin has two choices:**

- **Verify** (green button) — *"Yes, I checked the bank statement and this payment is real!"*
  - The payment amount gets added to "Amount Paid"
  - If the full amount is paid, status changes to **Paid**
  - If only part is paid, status changes to **Partial**
  - The parent gets a notification: *"Your payment has been verified!"*

- **Reject** (red button) — *"I couldn't find this transaction in our bank records"*
  - The admin writes a reason (e.g., "UTR number not found in bank statement")
  - The payment goes back to **Pending**
  - The parent gets a notification: *"Your payment was not verified. Reason: ..."*
  - The parent can try submitting again with correct details

### Online Payments — The Easy Mode

When a parent pays **online** (through Zaakpay, credit card, UPI on the website):
- The system records the payment **automatically**
- No admin verification needed!
- The payment goes straight from **Pending** to **Paid**
- It's like a self-checkout at a supermarket — the machine handles everything

### Understanding Payment Statuses

Here's a cheat sheet for what each status means:

| Status | What It Means | Color |
|--------|--------------|-------|
| **Pending** | Not paid yet. The student still owes this money. | Yellow |
| **Pending Review** | The parent says they paid, but the admin hasn't checked yet. | Blue |
| **Partial** | Some money was paid, but not the full amount. | Yellow |
| **Paid** | All done! The full amount has been received. | Green |
| **Overdue** | The due date has passed, and the fee hasn't been paid. Uh oh! | Red |
| **Exempt** | Free pass! No payment needed. | Blue |

### Payment Methods the System Supports

| Method | How It Works |
|--------|-------------|
| **Cash** | Parent hands over cash at the school office |
| **UPI** | Instant transfer using apps like Google Pay, PhonePe |
| **Bank Transfer** | NEFT / RTGS / IMPS transfer to school's bank account |
| **Cheque** | Parent writes a cheque |
| **Card** | Credit or debit card payment |
| **Online** | Paid through the website checkout (WooCommerce + payment gateway) |

---

## Part 5: Paying Fees — The Parent's Way (Frontend)

> *This section is written for you to show your parents! It explains how they can view and pay fees from the school's website.*

### Step 1: Log In to the School Website

1. Open the school's website in a browser
2. Click **Log In** (or go to "My Account" if it's a WooCommerce site)
3. Enter username and password
4. You're in!

### Step 2: Go to the Fees Page

After logging in, look for a **"Fees"** page or a **"Finance Fees"** section in your account dashboard.

**What you'll see:**

- **Your Info** at the top: Name, Grade, Division, Academic Year
- **Fee Cards** — one card for each fee. Each card shows:
  - Fee name (e.g., "Tuition Fee")
  - Amount due (e.g., ₹18,000)
  - Due date (e.g., "Jun 15, 2025")
  - How much you've already paid
  - Status badge (Pending, Paid, Overdue, etc.)
  - A "Pay" button (if you still owe money)
- **Summary** at the bottom:
  - Total Fees: ₹28,000
  - Total Paid: ₹5,000
  - Balance Due: ₹23,000

### Method A: Pay Online (Recommended)

This is the fastest way. The payment is recorded automatically — no waiting!

1. Find the fee card you want to pay (e.g., "Tuition Fee")
2. Click the **"Pay Now"** button
   - Or click **"Pay All Dues"** at the bottom to pay everything at once
3. You'll be taken to a **checkout page** (like shopping online)
4. Choose your payment method:
   - Credit/Debit Card
   - UPI
   - Net Banking
   - Wallets (if available)
5. Complete the payment
6. You'll be redirected back to the fees page
7. The fee card now shows a green **"Paid"** badge!
8. You'll also receive a **confirmation notification** (email/WhatsApp/SMS)

### Method B: Pay Offline (Bank Transfer, Cash, UPI outside the site)

Use this if you already paid via bank transfer, UPI app, or cash at the school.

1. Find the fee card you want to mark as paid
2. Click **"Submit Offline Payment"**
3. An inline form appears with these fields:

   | Field | What to Enter | Required? |
   |-------|--------------|-----------|
   | Amount | Pre-filled with balance due | (display only) |
   | Payment Method | Choose from dropdown (UPI, Bank Transfer, Cash, etc.) | Yes |
   | Reference/UTR Number | The transaction ID from your bank/UPI app | Yes |
   | Payment Date | When you made the payment | Yes (defaults to today) |
   | Receipt Screenshot | Upload a photo/PDF of your payment receipt | No (but helpful!) |
   | Notes | Any extra info for the school | No |

4. Click **Submit**
5. You'll see a success message: *"Payment details submitted! Pending verification by the administrator."*
6. The fee card now shows a blue **"Pending Verification"** badge
7. The school admin will review and verify your payment
8. Once verified, you'll get a notification and the badge turns green: **"Paid"**

### Method C: Direct Payment Links

Sometimes the school sends you a **payment link** via WhatsApp or SMS. It looks like:

```
https://school-website.com/fees/pay/123/
```

When you click it:
1. If you're not logged in, you'll be asked to log in first
2. If online payments are enabled, you go straight to checkout
3. If online payments are off, you're taken to the fees page with the offline form open
4. Complete the payment using Method A or B above

### How to View a Receipt

After a payment is verified (either automatically for online, or by the admin for offline):
1. Go to the Fees page
2. Find the fee card that shows **"Paid"**
3. The "Paid" badge is clickable — click on it!
4. A receipt opens in a new tab that you can print or save

---

## Part 6: The Journey of a Payment

Here's how a payment travels through the system from start to finish:

### Path A: Online Payment (The Fast Lane)

```
Parent clicks "Pay Now"
       |
       v
  Checkout page opens
       |
       v
  Parent completes payment
  (card, UPI, net banking)
       |
       v
  System AUTOMATICALLY records it
  (no waiting, no admin needed)
       |
       v
  Status: PAID
       |
       v
  Parent gets confirmation
  (email / WhatsApp / SMS)
```

### Path B: Offline Payment (The Verified Lane)

```
Parent pays via bank transfer,
cash, or UPI app
       |
       v
Parent goes to school website
and submits payment details
(UTR, date, receipt)
       |
       v
Status: PENDING REVIEW
(waiting for admin to check)
       |
       +----------+----------+
       |                     |
       v                     v
  Admin VERIFIES         Admin REJECTS
  (payment is real)      (can't find it)
       |                     |
       v                     v
  Status: PAID          Status: PENDING
       |                 (parent can try
       v                  again with
  Parent notified        correct details)
  "Payment verified!"        |
                             v
                        Parent notified
                        "Payment rejected.
                         Reason: ..."
```

### Path C: Overdue (The Reminder Lane)

```
Fee has a due date (e.g., Jun 15)
       |
       v
  System sends EARLY REMINDER
  before the due date
       |
       v
  Due date arrives...
  Did the parent pay?
       |
       +--------+---------+
       |                  |
       v                  v
    YES: Paid          NO: Not paid
    (all good!)           |
                          v
                    Status: OVERDUE
                          |
                          v
                    System sends
                    OVERDUE REMINDER
                          |
                          v
                    Parent pays
                    (online or offline)
                          |
                          v
                    Status: PAID
```

---

## Quick Glossary

| Term | Simple Meaning |
|------|---------------|
| **Academic Year** | The school year, e.g., "2025-2026" |
| **Grade** | The class level (Grade 1, Grade 5, etc.) |
| **Division** | The section within a grade (A, B, C) |
| **Fee Matrix** | The master price list of all fees |
| **Slab** | A single fee category (Tuition, Transport, Lab, etc.) |
| **Enrollment** | Registering a student for a specific year, grade, and division |
| **Payment Record** | A single fee entry for a student (e.g., "Riya owes ₹18,000 for Tuition") |
| **Transaction** | The actual act of paying money — with details like method, reference number, date |
| **Verification** | The admin checking that an offline payment is real |
| **Exempt** | A student who doesn't have to pay (RTE / scholarship) |
| **UTR** | Unique Transaction Reference — the ID number your bank gives when you transfer money |
| **WooCommerce** | A popular online shopping plugin that handles the checkout and payment gateway |
| **Zaakpay** | A payment gateway (like a digital cash register) that processes card/UPI payments |

---

## Remember!

- **Online payments** are instant and automatic — no admin work needed
- **Offline payments** need the admin to verify before they count
- The **Fee Matrix** is the source of truth — change it, and all payments update automatically
- **Enrolling** a student is the trigger that creates all their fee records
- Every payment has a **status** that tells you exactly where it stands
- Parents can always check their fees online — no need to call the school!

---

*Guide version: 1.0 | For qTap Finance v3.0.7+*
*Written with care for curious minds of all ages.*
