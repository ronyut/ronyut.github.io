---
title: "The 1-Shekel Ticket: Broken Access Control, Mass PII Exposure, and Price Manipulation in a Concert Ticketing Platform"
slug: "barby-ticketing-idor-price-manipulation"
date: 2024-09-17
description: "A security research case study analyzing broken access control and business logic vulnerabilities in an Israeli concert ticketing platform, resulting in mass PII exposure, account takeover, and arbitrary price manipulation."
tags: ["IDOR", "Broken Access Control", "Business Logic", "PII Exposure", "Web Security"]
categories: ["Security Research"]
featured_image: "featured.png"
---

## Executive Summary

During a routine ticket purchase on **Barby.co.il**, a major Tel Aviv-based concert venue and ticketing platform, I identified multiple high-impact security vulnerabilities caused by broken access control and improper trust boundary enforcement.

The issues enabled:
- Unauthenticated access to sensitive customer orders
- Mass exposure of personally identifiable information (PII)
- Full account takeover without prior authentication
- Arbitrary ticket price manipulation, including purchases for as little as **1 ILS**

The exposed data included full names, phone numbers, email addresses, partial credit card metadata, and internal order artifacts. In several cases, API responses also returned Israeli national ID numbers.

All findings were responsibly disclosed to Israel National Cyber Directorate (INCD).

---

{{< alert icon="shield" cardColor="#e2131380" iconColor="#f1faee" >}}
**Vulnerability Risk Assessment: Critical\***

- **Affected Platform**: Barby.co.il (ticketing & payment platform)
- **Impact**: Mass PII exposure (estimated ~100K clients), account takeover, financial fraud
- **Authentication Required**: None
- **Attack Complexity**: Low
- **Disclosure Status**: Reported to INCD
- **Remediation Status**: Platform claimed remediation
{{< /alert >}}
 
> \* This severity assessment reflects the combined impact of unauthenticated access control failures that enable cross-user data exposure, account takeover, and financial fraud. Individual issues may score lower in isolation. All core vulnerabilities are remotely exploitable, unauthenticated, and automatable. No victim interaction is required.

---

## Threat Model & Attack Surface

The platform exposed a broad unauthenticated attack surface:

- Public API endpoints returning sensitive order data
- Client-controlled pricing logic
- Predictable, sequential resource identifiers
- Account operations without authentication
- Direct access to ticket artifacts (PDFs, QR codes)

These findings indicate systemic access control failures rather than isolated bugs.

---

## Technical Findings

### 1. Insecure Direct Object Reference (IDOR) — Mass PII Exposure

**Endpoint**  
`GET /api/url/get-order/<ORDER_ID>`

**Authentication**: None

Order records were retrievable using sequential numeric identifiers. Incrementing the identifier allowed unrestricted enumeration of customer orders.

**Exposed Fields**
- Full name
- Phone number
- Email address
- Credit card last four digits and expiration
- Ticket PDF filename

**Impact**

Bulk harvesting of customer data was possible and fully automatable.

---

### 2. Unauthenticated Password Reset → Full Account Takeover

**Endpoint**  
`POST /api/users/change-password-n`

**Authentication**: None

The endpoint accepted only an email address and a new password, without verifying ownership.

**Exploit**
```http
POST /api/users/change-password-n HTTP/1.1
Content-Type: application/json

{
   "email": "victim@example.com",
   "newPassword": "attacker_password"
}
```

**Response Issues**
- Immediate password change
- JWT token returned to attacker
- Exposure of national ID number (`tz`)

**Attack Chain**
1. Email harvested via IDOR
2. Password reset without authentication
3. Full account access granted

---

### 3. Client-Side OTP Generation

**Source File**: `UserProfile.jsx`

```javascript
let _randomCode = generateCode(5);
await axios.post(`${BASE_URL}/api/users/reset-password`, { 
  postUserInfo, 
  _randomCode 
});
```

The One-Time Password (OTP) was generated client-side and submitted to the backend, rendering it non-secret.

In practice, this control provided no additional security and could be bypassed entirely without interacting with the OTP flow.

---

### 4. Direct Access to Ticket PDFs

**Endpoint**  
`GET /api/post-pdf/<TICKET_FILENAME>`

**Authentication**: None

Ticket PDFs were directly accessible. When combined with the IDOR exposure of filenames, attackers could download valid tickets belonging to other users.

---

### 5. Server-Side Price Integrity Failure

Ticket pricing was determined client-side and trusted by the backend during checkout.

**Impact**

- Arbitrary ticket pricing
- Direct financial loss

**Proof of Concept**

A transaction was successfully completed for 1 ILS, producing a valid ticket PDF.

![Payment page rendering attacker-controlled ticket price of 1 ILS](media/payment.png?w=450 "Payment page rendering attacker-controlled ticket price")

Redacted request to `POST /api/genlink`:

```json
{
  "amount": 0.01,
  "price": "0.01",
  "show_id": "4264",
  "status": "1",
  "...": "[redacted]"
}
```

Redacted response:

```json
{
  "results": { "status": "success" },
  "data": {
    "payment_page_link": "https://payments.[redacted]/[uuid]"
  }
}
```

This confirms that the backend:

- Accepts client-submitted prices
- Persists manipulated transaction values
- Generates valid payment links using poisoned data


**Root Cause**

The backend failed to recompute or validate pricing against authoritative server-side data before generating a payment request.

#### Clarification: Payment Link Generation Responsibility

The payment service provider is not responsible for this vulnerability.

Payment links are generated by the Barby backend, which submits attacker-controlled pricing values to the payment API. The payment provider processes the request as received, without independent business logic validation.

---

## Root Cause Analysis

1. **Broken Trust Boundaries**  
   Client input was treated as authoritative.

2. **Missing Authorization Controls**  
   Sensitive endpoints lacked authentication checks.

3. **Predictable Identifiers**  
   Sequential IDs enabled enumeration.

4. **Security Logic in the Browser**  
   OTPs and pricing must be enforced server-side.

---

## Remediation Recommendations

### Immediate

- Enforce server-side price calculation
- Require authentication for sensitive endpoints
- Replace sequential IDs with UUIDs
- Cryptographically sign QR codes
- Fix password reset flow with ownership validation

### Long-Term

- Conduct full security audit
- Implement rate limiting and monitoring
- Train developers on OWASP Top 10
- Establish secure SDLC practices

---

## Disclosure Timeline

- **July 14, 2024** – Vulnerabilities discovered
- **July 16, 2024** – Consulted with Check Point Research
- **July 18, 2024** – Report submitted to INCD
- **September 2024** – Platform claimed remediation

---

## Conclusion

This research demonstrates how broken access control and misplaced trust can collapse an entire security model.  
For platforms handling payments and PII, security must be enforced at the architectural level—not retrofitted after deployment.

If you’d like to discuss this research or related security topics, you can reach me on [LinkedIn](https://www.linkedin.com/in/rony-utevsky/).
