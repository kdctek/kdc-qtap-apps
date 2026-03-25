# qTap Finance — A Beginner's Guide

> *Imagine you're the principal of a school. Hundreds of students come every year, and each one has to pay different fees. How do you keep track of who paid what, who still owes money, and who gets a free pass? That's exactly what **qTap Finance** does — it's your digital accountant!*

---

## Table of Contents

1. [What is qTap Finance?](#part-1-what-is-qtap-finance)
2. [The Fee Matrix — Your Price List](#part-2-the-fee-matrix--your-price-list)
3. [Academic Terms — Splitting the Year](#part-3-academic-terms--splitting-the-year)
4. [Enrollments — Giving Each Student Their Fee Sheet](#part-4-enrollments--giving-each-student-their-fee-sheet)
5. [Adjustments — Discounts & Surcharges](#part-5-adjustments--discounts--surcharges)
6. [Multi-Year Programs](#part-6-multi-year-programs)
7. [Recording Payments — The Accountant's Way (Backend)](#part-7-recording-payments--the-accountants-way-backend)
8. [Paying Fees — The Parent's Way (Frontend)](#part-8-paying-fees--the-parents-way-frontend)
9. [The Journey of a Payment](#part-9-the-journey-of-a-payment)

---

## Part 1: What is qTap Finance?

Think of your school like a small business:

| Real World | In qTap Finance |
|-----------|-----------------|
| The school year (April 2025 to March 2026) | **Academic Year** (2025-2026) |
| A 2-year diploma program (2025 to 2027) | **Multi-Year** (2025-2027) |
| Classes like "Grade 1", "Grade 5" | **Grades** |
| Sections like "A", "B", "C" | **Divisions** |
| A list of all fees and their prices | **Fee Matrix** |
| Term 1, Term 2, Term 3 | **Academic Terms** |
| Writing a student's name in the register | **Enrollment** |
| The bill you get from a shop | **Payment Record** |
| The receipt after you pay | **Transaction** |

### The Building Blocks

Everything in qTap Finance is built on these things:

1. **Academic Years** — The school calendar. Example: "2025-2026" (standard) or "2025-2027" (multi-year for 2-year programs).

2. **Grades & Divisions** — How your school organizes students. Grades are the class level (Nursery, LKG, Grade 1...), and Divisions are the sections within each grade (A, B, C...).

3. **Fee Matrix** — The master price list with **fee types** that control how charges are calculated:
   - **Per Month** — charged every month (e.g., transport)
   - **Per Term** — charged once per academic term (e.g., exam fee)
   - **Per Cycle** — charged once per 12-month cycle (e.g., annual maintenance)
   - **Per Tenure** — charged once for the entire program (e.g., admission fee)

4. **Academic Terms** — How the year is divided (Term 1: Apr-Jul, Term 2: Aug-Nov, Term 3: Dec-Mar). Each term has its own months and due date.

5. **Enrollments** — The act of saying: *"This student is in Grade 5, Division A, for the year 2025-2026."* The moment you enroll a student, the system automatically creates all their fee records.

6. **Adjustments** — Per-student discounts or surcharges. *"This student gets a 10% discount on Tuition Fee."*

---

## Part 2: The Fee Matrix — Your Price List

### What is a "Slab"?

A **slab** is a **fee category** — a type of fee you charge students.

**Common examples of slabs:**

| Slab Name | What It's For |
|-----------|---------------|
| Tuition Fee | The main school fee for teaching |
| Transport Fee | School bus charges |
| Lab Fee | Science/Computer lab equipment |
| Activity Fee | Sports, music, art activities |
| PTA Contribution | Parent-Teacher Association fund |

### Fee Types — How Charges Are Calculated

Each slab has up to four **fee types** that control how amounts are calculated:

| Fee Type | When It's Charged | Example |
|----------|------------------|---------|
| **Per Month** | Every single month | Transport: ₹1,000/month |
| **Per Term** | Once per academic term | Exam Fee: ₹500/term |
| **Per Cycle** | Once per 12-month cycle | Lab Maintenance: ₹2,000/year |
| **Per Tenure** | Once for the entire program | Admission Fee: ₹10,000 (one-time) |

A single slab can use multiple fee types! For example, "Tuition Fee" might have both a Per Month component (₹2,000/month) and a Per Tenure component (₹5,000 one-time admission).

### Collection Modes — How Often Students Pay

Each slab also has a **collection mode** that determines how payments are grouped:

| Mode | What It Means |
|------|--------------|
| **Monthly** | One payment every month |
| **Quarterly** | One payment every 3 months |
| **Half Yearly** | One payment every 6 months |
| **Full Year** | One payment per 12-month cycle |
| **Full Tenure** | One single payment for everything |

### How Amounts Differ by Grade

The **same slab** can have **different prices** for different grades:

**Example Fee Matrix for 2025-2026:**

| Slab | Fee Type | Nursery | Grade 1 | Grade 5 | Grade 10 |
|------|----------|---------|---------|---------|----------|
| Tuition Fee | Per Term | ₹4,000 | ₹5,000 | ₹6,000 | ₹8,000 |
| Transport Fee | Per Month | ₹1,000 | ₹1,000 | ₹1,000 | ₹1,000 |
| Lab Fee | Per Cycle | — | — | ₹3,000 | ₹5,000 |
| PTA | Per Tenure | ₹2,000 | ₹2,000 | ₹2,000 | ₹2,000 |

Notice how:
- Tuition increases as the grade goes up (charged per term)
- Transport is the same for everyone (charged monthly)
- Lab Fee only applies to Grade 5+ (per cycle — once per year)
- PTA is a one-time charge (per tenure)

### What About Free Passes? (Exempt Fees)

Some students get **exemptions** — they don't have to pay certain fees. When a slab is marked as "Can be Exempted," the admin can give specific students a free pass. Their payment record will show **"Exempt"** instead of a pending amount.

### Fee Sort Order

Admins can control how fee line items are ordered in payments:
- **Fee type first** — Groups all Per Tenure items together, then Per Cycle, Per Term, Per Month
- **Slab position first** — Shows items in the order slabs appear in the Fee Matrix

The order of fee types themselves is also drag-to-reorder (Per Tenure → Per Cycle → Per Term → Per Month by default).

### Step-by-Step: How an Admin Sets Up the Fee Matrix

**Step 1: Set up the basics** (General Tab)

1. Go to **qTap Finance** in the admin menu
2. Click the **General** tab
3. Add your **Academic Years** (e.g., "2025-2026")
4. Add your **Grades** (e.g., Nursery, LKG, UKG, Grade 1...)
5. Add your **Divisions** (e.g., A, B, C, D)
6. Add your **Fee Categories** — these become your slabs
7. Click **Save**

**Step 2: Set the current year** (Institute Tab)

1. Click the **Institute** tab
2. Select the **Current Academic Year** (for standard years)
3. Select the **Current Multi-Year** (if you have multi-year programs)
4. Fill in institute details and currency settings
5. Click **Save**

**Step 3: Set up Academic Terms** (General Tab)

1. Go back to the **General** tab
2. In the **Academic Terms** section, create terms (e.g., Term 1, Term 2, Term 3)
3. For each term, select which months belong to it
4. Set the **due date** for each term
5. Click **Save Terms**

**Step 4: Build the fee matrix** (Fee Matrix Tab)

1. Click the **Fee Matrix** tab
2. Select the academic year from the dropdown
3. For each fee slab:
   - Choose the **Collection Mode** (how often to collect)
   - For each **Fee Type** row (Per Month, Per Term, Per Cycle, Per Tenure):
     - Enter the amount for each grade
     - Check the "Enable" checkbox for grades that should pay
   - Check **"Exempt"** if some students might get a free pass
4. Click **Save Fee Matrix**

---

## Part 3: Academic Terms — Splitting the Year

### Why Terms Matter

Instead of one giant bill at the start of the year, schools typically split fees into **terms**. Each term covers a portion of the academic year.

**Example for 2025-2026 (April start):**

| Term | Months | Due Date |
|------|--------|----------|
| Term 1 | Apr, May, Jun, Jul | April 15, 2025 |
| Term 2 | Aug, Sep, Oct, Nov | August 15, 2025 |
| Term 3 | Dec, Jan, Feb, Mar | December 15, 2025 |

### How Terms Affect Payments

When a student is enrolled, the system uses terms to create billing periods:

- **Per Month** items create one line item per month within each billing period
- **Per Term** items create one line item per term
- **Per Cycle** items create one line item per 12-month cycle
- **Per Tenure** items create one line item for the entire program (attached to the first billing period)

**Example: Grade 5 student with Quarterly collection mode:**

| Payment | Per Month (Transport ₹1,000) | Per Term (Exam ₹500) | Per Tenure (PTA ₹2,000) | Total |
|---------|------------------------------|---------------------|------------------------|-------|
| Q1 (Apr-Jun) | ₹3,000 (3 months) | ₹500 (Term 1) | ₹2,000 | ₹5,500 |
| Q2 (Jul-Sep) | ₹3,000 (3 months) | — | — | ₹3,000 |
| Q3 (Oct-Dec) | ₹3,000 (3 months) | ₹500 (Term 2) | — | ₹3,500 |
| Q4 (Jan-Mar) | ₹3,000 (3 months) | ₹500 (Term 3) | — | ₹3,500 |

---

## Part 4: Enrollments — Giving Each Student Their Fee Sheet

### What Happens When You Enroll a Student?

Enrolling a student is like writing their name in the attendance register — but smarter:

1. You tell it: *"Riya is in Grade 5, Division A, for 2025-2026"*
2. The system looks at the Fee Matrix and finds all fees that apply to Grade 5
3. It uses the Collection Mode and Terms to create **billing period payments**
4. Each payment has granular line items (per_month, per_term, per_cycle, per_tenure)

### Payment Cycle Override

Each student can have a **payment cycle override** on their enrollment. This lets you change how often a specific student pays, independent of the slab's collection mode:

| Payment Cycle | How It Works |
|--------------|-------------|
| Monthly | 12 payments per year |
| Quarterly | 4 payments per year |
| Half Yearly | 2 payments per year |
| Full Year | 1 payment per 12-month cycle |
| Full Tenure | 1 payment for everything |

### Force Full Academic Year

Admins can enable **"Force full academic year payment"** per academic year. When enabled, all terms must be paid together as a single payment (unless a student has an individual payment cycle override).

---

## Part 5: Adjustments — Discounts & Surcharges

### What Are Adjustments?

Adjustments let admins give individual students **discounts** or **surcharges** on specific fee slabs.

**Examples:**
- *"Riya gets a 10% discount on Tuition Fee"* (percentage discount)
- *"Aman gets ₹500 off on Transport Fee"* (fixed discount)
- *"Priya has an extra ₹1,000 surcharge for special coaching"* (surcharge)

### How Adjustments Work

Each adjustment has:
- **Label** — A name (e.g., "Sibling Discount", "Sports Scholarship")
- **Type** — Discount or Surcharge
- **Mode** — Fixed amount (₹500) or Percentage (10%)
- **Amount** — The value
- **Target Slabs** — Which fee slabs it applies to (checkboxes)
- **Show Badge** — Whether it appears as a visible line item or is silently applied

### Badge ON vs Badge OFF

| Setting | What Happens |
|---------|-------------|
| **Badge ON** | The adjustment appears as a **separate line item** in payments (e.g., "Sibling Discount: -₹500"). The original slab shows its full price. |
| **Badge OFF** | The adjustment is **silently baked into** the slab amounts. The parent sees the adjusted price directly — no separate discount line. |

Both produce the same net total — the difference is just how it's displayed.

### Adjustments in Enrollment Cards

In the admin's enrollment view, adjustments show as colored badges:
- Discounts: `-10%` or `-₹500` with an arrow pointing to the target slab
- Surcharges: `+₹1,000` with the target slab name

---

## Part 6: Multi-Year Programs

### What Are Multi-Year Academic Years?

Some programs span more than one year — like a 2-year diploma or a 3-year degree. qTap Finance supports this with **multi-year academic years**.

**Examples:**
- `2025-2026` — Standard 1-year (12 months)
- `2025-2027` — 2-year program (24 months, 2 cycles)
- `2025-2028` — 3-year program (36 months, 3 cycles)

### How Cycles Work

Each 12-month slice of a multi-year program is called a **cycle**:
- `2025-2027` has **Cycle 1** (2025-2026) and **Cycle 2** (2026-2027)
- `2025-2028` has **Cycle 1**, **Cycle 2**, and **Cycle 3**

Fee types interact with cycles:
- **Per Month** — charged every month across all cycles (24 charges for a 2-year program)
- **Per Term** — charged per term across all cycles
- **Per Cycle** — charged **once per cycle** (2 charges for a 2-year program)
- **Per Tenure** — charged **once for the entire program** (1 charge total)

### Multi-Year in the Terms UI

When setting up terms for a multi-year program, the month checkboxes show grouped by cycle:

```
Year 1 (2025-2026): [Apr] [May] [Jun] ... [Mar]
Year 2 (2026-2027): [Apr] [May] [Jun] ... [Mar]
```

### Current Year Settings

The Institute tab has **two** dropdowns:
- **Current Academic Year** — For standard 1-year programs
- **Current Multi-Year** — For multi-year programs

When the system needs to determine which year to show for a student, **multi-year enrollment takes priority**. If a student is enrolled in both "2025-2026" and "2025-2027", the multi-year one is shown first.

---

## Part 7: Recording Payments — The Accountant's Way (Backend)

### The Admin's Dashboard

When the admin logs into WordPress, they go to **qTap Finance** and see several tabs. For payments, the most important one is:

**Verify Payments** — The admin's inbox. Whenever a parent says *"I paid!"*, it shows up here for the admin to check and approve.

### How the Verification Process Works

1. Riya's dad transfers ₹18,000 to the school's bank account
2. He submits the payment details on the website (UTR number, date, screenshot)
3. The payment shows up in the admin's **Verify Payments** tab

**The admin has two choices:**

- **Verify** — *"Yes, this payment is real!"*
  - Amount gets added to "Amount Paid"
  - Status changes to **Paid** (or **Partial** if not fully paid)
  - Parent gets a notification

- **Reject** — *"I couldn't find this transaction"*
  - Admin writes a reason
  - Payment goes back to **Pending**
  - Parent gets a notification with the reason

### Online Payments — The Easy Mode

When a parent pays **online** (through the payment gateway):
- The system records the payment **automatically**
- No admin verification needed!
- Payment goes straight from **Pending** to **Paid**

### Understanding Payment Statuses

| Status | What It Means | Color |
|--------|--------------|-------|
| **Pending** | Not paid yet | Yellow |
| **Pending Review** | Parent says they paid, admin hasn't checked | Blue |
| **Partial** | Some money paid, not the full amount | Yellow |
| **Paid** | All done! Full amount received | Green |
| **Overdue** | Due date passed, fee unpaid | Red |
| **Exempt** | Free pass! No payment needed | Blue |

---

## Part 8: Paying Fees — The Parent's Way (Frontend)

### Step 1: Log In to the School Website

1. Open the school's website
2. Click **Log In**
3. Enter username and password

### Step 2: Go to the Fees Page

After logging in, look for a **"Fees"** page. You'll see:

- **Your Info** at the top: Name, Grade, Division, Academic Year
- **Fee Cards** — one for each billing period, with line items showing:
  - Fee name and type (e.g., "Tuition Fee - Per Term")
  - Amount due, amount paid, balance
  - Due date and status badge
  - A "Pay" button
- **Summary** at the bottom: Total Fees, Total Paid, Balance Due

### Method A: Pay Online (Recommended)

1. Click **"Pay Now"** on a fee card (or **"Pay All Dues"** at the bottom)
2. Choose payment method (Card, UPI, Net Banking)
3. Complete the payment
4. Fee card shows green **"Paid"** badge!

### Method B: Pay Offline

1. Click **"Submit Offline Payment"**
2. Fill in: Payment Method, Reference/UTR Number, Date, Receipt Screenshot
3. Click **Submit**
4. Badge shows **"Pending Verification"** until admin verifies

### WhatsApp Fee Lookup

Schools can integrate with WhatsApp to let parents check fees via chat. The system provides an API that returns pending payments in WhatsApp's interactive list format — parents can view and pay directly from WhatsApp.

---

## Part 9: The Journey of a Payment

### Path A: Online Payment (The Fast Lane)

```
Parent clicks "Pay Now"
       → Checkout page opens
       → Parent completes payment
       → System AUTOMATICALLY records it
       → Status: PAID
       → Parent gets confirmation
```

### Path B: Offline Payment (The Verified Lane)

```
Parent pays via bank/UPI/cash
       → Submits details on website
       → Status: PENDING REVIEW
       → Admin VERIFIES or REJECTS
       → If verified: PAID + notification
       → If rejected: PENDING + reason sent
```

### Path C: Overdue (The Reminder Lane)

```
Fee has a due date
       → System sends EARLY REMINDER
       → Due date arrives...
       → If paid: All good!
       → If not: Status: OVERDUE
       → System sends OVERDUE REMINDER
       → Parent pays → Status: PAID
```

---

## Quick Glossary

| Term | Simple Meaning |
|------|---------------|
| **Academic Year** | The school year, e.g., "2025-2026" |
| **Multi-Year** | A program spanning multiple years, e.g., "2025-2027" |
| **Cycle** | A 12-month slice of a multi-year program |
| **Grade** | The class level (Grade 1, Grade 5, etc.) |
| **Division** | The section within a grade (A, B, C) |
| **Fee Matrix** | The master price list of all fees |
| **Slab** | A single fee category (Tuition, Transport, Lab, etc.) |
| **Fee Type** | How a charge is calculated: Per Month, Per Term, Per Cycle, or Per Tenure |
| **Collection Mode** | How often payments are grouped: Monthly, Quarterly, Half Yearly, Full Year, Full Tenure |
| **Academic Term** | A chunk of the year (Term 1, Term 2) with its own months and due date |
| **Enrollment** | Registering a student for a specific year, grade, and division |
| **Adjustment** | A per-student discount or surcharge on specific slabs |
| **Payment Record** | A billing period entry for a student with line items |
| **Transaction** | The actual act of paying money — with method, reference number, date |
| **Payment Cycle** | Per-student override of how billing periods are grouped |
| **Verification** | The admin checking that an offline payment is real |
| **Exempt** | A student who doesn't have to pay (RTE / scholarship) |
| **UTR** | Unique Transaction Reference — the ID from your bank |
| **WooCommerce** | Online shopping plugin that handles checkout and payment gateway |

---

## Remember!

- **Online payments** are instant and automatic — no admin work needed
- **Offline payments** need the admin to verify before they count
- The **Fee Matrix** is the source of truth — with granular fee types for flexible billing
- **Academic Terms** define the billing structure and due dates
- **Enrolling** a student is the trigger that creates all their fee records
- **Adjustments** let you give individual discounts or add surcharges
- **Multi-year programs** work just like standard years, with Per Cycle and Per Tenure handling the billing correctly
- Every payment has a **status** that tells you exactly where it stands
- Parents can always check their fees online — no need to call the school!

---

*Guide version: 2.0 | For qTap Finance v3.10.9+*
*Written with care for curious minds of all ages.*
