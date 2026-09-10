# Sensitive Information Disclosure Through API Response

**Severity:** Medium
**Category:** Information Disclosure / API Security
**CWE:** CWE-200 — Exposure of Sensitive Information to an Unauthorized Actor
**Status:** Remediated
**Environment:** Web application / REST API
**Testing Type:** Authorized black-box security assessment

> **Disclaimer:** This report has been anonymized for portfolio purposes. Target domains, identifiers, usernames, tokens, internal hostnames, and sensitive response data have been replaced or removed.

---

## 1. Executive Summary

During an authorized black-box assessment, I identified an information-disclosure vulnerability in a REST API endpoint.

The endpoint returned internal account-related information that was not required for the application's normal client-side functionality.

The disclosed information included internal object identifiers and additional metadata associated with the requested resource.

Although the exposed information did not directly provide authentication credentials, it increased the amount of information available to an unauthorized or low-privileged user and could assist further reconnaissance or exploitation of other application functionality.

---

## 2. Affected Functionality

The affected functionality was an API endpoint used to retrieve account/resource information.

Example:

```text
GET /api/profile
```

The endpoint was accessible to an authenticated low-privileged user.

---

## 3. Preconditions

The tester required:

* A normal application account
* No administrative privileges
* No special application permissions

No server compromise or elevated privileges were required.

---

## 4. Reproduction Steps

### Step 1 — Authenticate normally

A test account was created and authenticated through the normal application workflow.

Sensitive authentication material has been redacted.

```http
POST /api/login HTTP/1.1
Host: target.example
Content-Type: application/json

{
  "email": "tester@example.invalid",
  "password": "[REDACTED]"
}
```

---

### Step 2 — Request the affected resource

The authenticated session was used to access the API endpoint:

```http
GET /api/profile HTTP/1.1
Host: target.example
Authorization: Bearer [REDACTED]
```

The server returned:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

with a response similar to:

```json
{
  "id": "usr_123456",
  "email": "tester@example.invalid",
  "name": "Test User",
  "internalRole": "user",
  "accountCreatedAt": "2026-01-15T12:30:00Z",
  "internalReference": "ACC-847291",
  "internalStatus": "ACTIVE"
}
```

The exact production values have been removed.

---

## 5. Security Issue

Several returned fields were not necessary for the intended client-side functionality.

For example:

```text
internalReference
internalRole
internalStatus
```

were exposed directly through the API.

The application therefore returned a larger data set than was necessary for the requesting client.

The principle involved is:

```text
Server-side data
       │
       ▼
Select only required fields
       │
       ▼
API response
```

rather than:

```text
Server-side object
       │
       ▼
Serialize entire object
       │
       ▼
API response
```

---

## 6. Security Impact

Depending on the nature of the exposed fields, excessive information disclosure can:

* Reveal internal identifiers
* Expose implementation details
* Help map application objects
* Assist enumeration of application resources
* Reveal internal account state
* Provide useful information for chaining with other vulnerabilities

The observed issue did **not** by itself demonstrate authentication bypass or account takeover.

The impact was therefore assessed based on the information actually exposed.

---

## 7. Root Cause

The likely root cause was insufficient response-data filtering.

The server appeared to serialize internal application data directly into the API response instead of explicitly selecting fields intended for client consumption.

A safer model is to define a dedicated response object:

```python
{
    "id": user.id,
    "name": user.name
}
```

rather than returning the complete internal database/application object.

---

## 8. Recommended Remediation

### Return only required fields

API responses should contain only information required by the client.

For example:

```json
{
  "id": "usr_123456",
  "name": "Test User"
}
```

instead of exposing internal metadata.

### Review API serialization

Review:

* REST endpoints
* GraphQL resolvers
* Debug endpoints
* Error responses
* Administrative APIs
* Object serialization
* Database-to-JSON conversion

### Apply authorization server-side

Removing fields from the UI is not sufficient.

Sensitive information must not be sent to the client in the first place.

---

## 9. Validation After Remediation

The same request was repeated after remediation.

Previously:

```json
{
  "id": "usr_123456",
  "email": "tester@example.invalid",
  "internalRole": "user",
  "internalReference": "ACC-847291"
}
```

After remediation:

```json
{
  "id": "usr_123456",
  "name": "Test User"
}
```

The unnecessary internal fields were no longer returned.

---

## 10. Evidence

Recommended sanitized evidence:

```text
evidence/
├── 01-request.txt
├── 02-original-response.json
└── 03-remediated-response.json
```

All sensitive values should be replaced with:

```text
[REDACTED]
```

or controlled test values.

---

## 11. Lessons Learned

API security testing should not focus exclusively on whether an endpoint is accessible.

The response itself must also be analyzed.

For every interesting endpoint, ask:

```text
Who can access it?
What data does it return?
Does the client actually need all of it?
Are internal identifiers exposed?
Are authorization decisions visible?
Are hidden fields actually hidden server-side?
Does changing the user's privilege change the response?
```

An endpoint can have correct authentication and authorization while still exposing more information than necessary.

---

## 12. Classification

**Primary:** CWE-200 — Exposure of Sensitive Information to an Unauthorized Actor

Additional classifications may be appropriate depending on the exact data exposed.

