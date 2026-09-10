# Business Logic Flaw Allows Reuse of Single-Use Discount

**Severity:** Medium
**Category:** Business Logic / Improper Validation
**CWE:** CWE-840 — Business Logic Errors
**Status:** Remediated
**Environment:** E-commerce web application
**Testing Type:** Authorized black-box security assessment

> **Disclaimer:** This report has been anonymized for portfolio purposes. The target application, discount codes, account information, transaction identifiers, and monetary values have been replaced with controlled test data.

---

## 1. Executive Summary

During an authorized assessment of an e-commerce application, I identified a business-logic flaw in the discount-code functionality.

A discount code intended to be usable only once per customer could be applied repeatedly by manipulating the order workflow.

The application validated the discount during checkout but failed to enforce the usage restriction consistently when subsequent orders were created.

As a result, a single-use promotional discount could be reused multiple times by the same account.

---

## 2. Intended Business Rule

The application's documented behavior was equivalent to:

```text
One customer
      │
      ▼
Redeem promotional code
      │
      ▼
Discount applied
      │
      ▼
Code becomes unavailable
```

Expected behavior:

```text
First order  → discount accepted
Second order → discount rejected
```

---

## 3. Preconditions

The tester required:

* A normal customer account
* A valid test promotional code
* Ability to create orders
* No administrative privileges

Testing was performed using controlled accounts and test transactions.

---

## 4. Reproduction Steps

### Step 1 — Create a test order

A controlled account was used to create an order.

Example:

```text
Account: tester@example.invalid
Cart value: $100
Discount: TEST10
```

The application correctly applied the discount.

```text
Original price: $100
Discount:       -$10
Final price:     $90
```

---

### Step 2 — Complete the order

The test order was completed through the normal application workflow.

The transaction was recorded successfully.

```text
Order ID: TEST-001
Status: completed
```

---

### Step 3 — Attempt to reuse the discount

A second order was created using the same account.

The same promotional code was submitted:

```http
POST /api/cart/apply-discount HTTP/1.1
Host: target.example
Authorization: Bearer [REDACTED]
Content-Type: application/json

{
  "code": "TEST10"
}
```

The application returned a successful response:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

Example:

```json
{
  "valid": true,
  "discount": 10
}
```

---

### Step 4 — Confirm the business rule violation

The second order received the same discount.

```text
Order 1
$100 → $90

Order 2
$100 → $90
```

The promotional code was therefore successfully reused despite being intended as a single-use promotion.

---

## 5. Security Impact

An attacker could repeatedly redeem a promotion that was intended to be limited to one use per customer.

Potential consequences include:

* Financial loss
* Abuse of promotional campaigns
* Incorrect revenue calculations
* Inventory/order abuse
* Increased operational costs
* Potential automated exploitation at scale

The exact financial impact depends on the value and restrictions associated with the promotional campaign.

---

## 6. Root Cause

The likely root cause was incomplete enforcement of the business rule.

The application correctly validated:

```text
Is the discount code valid?
```

but failed to consistently validate:

```text
Has this customer already used this discount?
```

A business rule should be enforced on the server at the point where the transaction is committed.

Client-side restrictions or temporary checkout state should not be trusted as the authoritative control.

---

## 7. Recommended Remediation

### Validate usage server-side

Before applying the discount:

```text
Is code valid?
        │
        ▼
Is code active?
        │
        ▼
Is customer eligible?
        │
        ▼
Has customer already used it?
        │
        ├── YES → Reject
        │
        └── NO → Apply
```

### Enforce the rule during order creation

The final order transaction should revalidate the promotion rather than trusting an earlier cart calculation.

### Prevent race conditions

If the promotion is limited to one use, the database operation should enforce the constraint atomically.

Conceptually:

```sql
BEGIN;

-- Verify eligibility
-- Reserve/redeem promotion
-- Create order

COMMIT;
```

The implementation should ensure that concurrent requests cannot cause multiple successful redemptions.

---

## 8. Validation After Remediation

The original workflow was repeated.

Expected result:

```text
First redemption:
TEST10 → accepted

Second redemption:
TEST10 → rejected
```

Example response:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
```

```json
{
  "error": "Promotion has already been used"
}
```

The business rule was therefore enforced server-side.

---

## 9. Evidence

Recommended sanitized evidence:

```text
evidence/
├── 01-first-redemption.txt
├── 02-first-order.txt
├── 03-second-redemption.txt
└── 04-remediation-verification.txt
```

Screenshots can demonstrate:

```text
Order #TEST-001 → discount applied

Order #TEST-002 → discount rejected
```

Use only controlled test transactions.

---

## 10. Why Automated Scanners May Miss This

This vulnerability illustrates an important difference between technical vulnerabilities and business-logic vulnerabilities.

A scanner can often identify:

```text
SQL injection
XSS
missing security headers
known vulnerable dependencies
```

But it is much harder for an automated scanner to understand:

```text
"This discount is supposed to be usable once per customer."
```

The tester must understand the application's intended workflow and test whether the implementation actually enforces that rule.

---

## 11. Testing Methodology

For business-logic testing, I recommend testing workflows rather than isolated endpoints.

For example:

```text
Register
   ↓
Login
   ↓
Add item
   ↓
Apply discount
   ↓
Checkout
   ↓
Complete order
   ↓
Repeat workflow
   ↓
Compare behavior
```

Useful questions include:

```text
Can an operation be repeated?

Can an operation be performed in an unexpected order?

Can client-controlled values influence server-side decisions?

Can a completed action be reversed?

Can a discount be reused?

Can quantities become negative or zero?

Can a payment state be manipulated?

Can two requests race each other?

Can an object transition into an invalid state?
```

---

## 12. Lessons Learned

Business-logic vulnerabilities require understanding what the application is **supposed to do**, not merely what its API allows.

A useful testing model is:

```text
Expected business rule
          │
          ▼
Actual server behavior
          │
          ▼
Find discrepancy
          │
          ▼
Determine security impact
          │
          ▼
Test remediation
```

The most valuable business-logic findings often come from deliberately performing legitimate actions in unusual sequences rather than sending obviously malicious payloads.

---

## 13. Classification

**Primary:** CWE-840 — Business Logic Errors

The exact CWE should be adjusted if the root cause is more specifically attributable to a transaction, race-condition, authorization, or input-validation weakness.

