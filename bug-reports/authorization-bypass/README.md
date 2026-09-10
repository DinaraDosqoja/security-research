# Missing Email Verification Enables Account Takeover

**Severity:** High
**Category:** Authentication / Account Takeover
**CWE:** CWE-287 — Improper Authentication
**Status:** Remediated
**Environment:** Web application
**Testing Type:** Authorized black-box security assessment

> **Disclaimer:** This report has been anonymized for portfolio purposes. Target domains, usernames, email addresses, tokens, identifiers, and other potentially sensitive information have been removed or replaced with placeholders.

---

## 1. Executive Summary

During an authorized black-box assessment of a web application, I identified an account takeover vulnerability caused by insufficient email verification during account registration and subsequent account-management operations.

The application allowed a newly registered account to authenticate without first verifying ownership of the supplied email address.

After registration, the email address associated with the account could be changed to an address controlled by the tester. The password could then be changed using the newly assigned email address.

This created a potential account takeover scenario because control of the original email address was not required to complete the account setup or modify critical authentication information.

---

## 2. Affected Functionality

The following account-management functionality was involved:

* User registration
* Login
* Email address modification
* Password modification

The vulnerable workflow can be summarized as:

```text
Register account
      │
      ▼
Email verification skipped
      │
      ▼
Login succeeds
      │
      ▼
Change email address
      │
      ▼
Change password
      │
      ▼
Account control obtained
```

---

## 3. Preconditions

The vulnerability could be reproduced using a normal unauthenticated user account.

No administrative privileges were required.

The tester only needed:

* Access to the public registration functionality
* An email address controlled by the tester
* An account created during testing

---

## 4. Reproduction Steps

### Step 1 — Register a new account

A new account was created using a controlled email address:

```text
Email: tester@example.invalid
Password: [REDACTED]
```

The application indicated that registration was successful.

Normally, an application requiring verified email ownership should prevent sensitive account operations until verification is completed.

---

### Step 2 — Verify whether email confirmation is enforced

No authorization/verification link was received.

Despite the absence of email verification, the newly registered account could authenticate successfully.

Example:

```http
POST /api/login HTTP/1.1
Host: target.example
Content-Type: application/json

{
  "email": "tester@example.invalid",
  "password": "[REDACTED]"
}
```

The server returned a successful authentication response.

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "authenticated": true
}
```

This established that email ownership was not required for authentication.

---

### Step 3 — Change the account email

While authenticated as the test account, the email address was changed to another address controlled by the tester.

Example:

```http
PATCH /api/account/email HTTP/1.1
Host: target.example
Authorization: Bearer [REDACTED]

{
  "email": "new-address@example.invalid"
}
```

The operation completed successfully.

No verification of the new email address was required before the address became associated with the account.

---

### Step 4 — Change the password

The password was then changed using the newly assigned email/account context.

Example:

```http
POST /api/account/password HTTP/1.1
Host: target.example
Authorization: Bearer [REDACTED]

{
  "newPassword": "[REDACTED]"
}
```

The password change succeeded.

---

## 5. Security Impact

The weakness could allow an attacker to gain control over an account if the application's account lifecycle permits an attacker-controlled email address to become the authoritative recovery/authentication address without proving ownership.

Potential consequences include:

* Unauthorized account access
* Account lockout of the legitimate user
* Password reset/control through an attacker-controlled email address
* Loss of account integrity
* Potential exposure of private user data
* Potential abuse of functionality available to the compromised account

The actual impact depends on the application's password-reset, session-management, and email-verification mechanisms.

---

## 6. Root Cause

The underlying issue appears to be insufficient enforcement of email ownership.

The application treated an email address supplied during registration or account modification as trusted authentication information without requiring successful verification.

A secure implementation should distinguish between:

```text
email supplied by user
        ≠
email verified by application
```

An email address should not become a trusted account-recovery or authentication factor until ownership has been established.

---

## 7. Recommended Remediation

### Enforce email verification

New accounts should remain in an unverified state until the user successfully completes the email-verification process.

For example:

```text
REGISTERED
    │
    ▼
EMAIL_UNVERIFIED
    │
    │ verification link
    ▼
EMAIL_VERIFIED
    │
    ▼
NORMAL ACCOUNT
```

### Protect email changes

Changing an account's email address should require appropriate verification.

A secure workflow could be:

```text
Authenticated user
       │
       ▼
Submit new email
       │
       ▼
Send verification link
       │
       ▼
User confirms new address
       │
       ▼
New email becomes verified
```

Depending on the application's threat model, changing the email should also require re-authentication or confirmation using the existing authentication factor.

### Protect password changes

Password changes should require appropriate authentication and should not rely solely on an unverified email address.

Additional controls may include:

* Current-password verification
* Recent authentication requirement
* MFA/re-authentication for sensitive operations
* Session invalidation after critical account changes
* Notifications when authentication information changes

---

## 8. Validation After Remediation

The fix should be tested using the same workflow.

Expected behavior:

```text
Register
   │
   ▼
Email verification required
   │
   ├── Not verified → sensitive operations denied
   │
   ▼
Verify email
   │
   ▼
Account becomes fully active
```

Similarly, changing the email address should not immediately make an unverified address authoritative.

---

## 9. Evidence

The original assessment contained HTTP requests and responses demonstrating the behavior.

For public disclosure, sensitive information should be replaced with placeholders:

```text
TARGET.example
tester@example.invalid
[REDACTED]
[REDACTED_TOKEN]
```

Recommended portfolio evidence:

```text
evidence/
├── 01-registration.txt
├── 02-login-before-verification.txt
├── 03-email-change.txt
├── 04-password-change.txt
└── 05-remediation-verification.txt
```

No real credentials, session tokens, customer information, or private target information should be included.

---

## 10. Lessons Learned

This assessment demonstrates why authentication testing should not stop at:

```text
Can I log in?
```

A more complete assessment asks:

```text
Can I register?
Can I authenticate before verification?
What does the application consider a trusted identity?
Can I change authentication attributes?
Is ownership verified?
Can I recover the account?
Are existing sessions invalidated?
Can I bypass security controls by changing account state?
```

Authentication vulnerabilities frequently arise from the interaction between multiple individually reasonable features rather than from a single obviously vulnerable endpoint.

---

## 11. Classification

**Primary category:** Authentication / Account Management

**Potential CWE:**

* CWE-287 — Improper Authentication
* CWE-306 — Missing Authentication for Critical Function
* CWE-640 — Weak Password Recovery Mechanism

The exact CWE should be selected based on the application's actual root cause and verified behavior rather than simply assigning every potentially relevant category.

---

## 12. Disclosure Note

This portfolio version intentionally excludes:

* Target domain
* Real usernames
* Real email addresses
* Session tokens
* Authentication headers
* Customer information
* Internal infrastructure details
* Exploit data that could identify the tested organization

The report demonstrates the testing methodology and security reasoning while preserving confidentiality.

