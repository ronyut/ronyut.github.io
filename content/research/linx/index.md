---
title: "The Trust Fall: Bypassing a City-Wide Payment Ecosystem"
slug: "linx-payment-bypass"
date: 2026-01-12
description: "What happens when a fintech platform trusts \"pixels on a screen\"? I audited the Linx payment flow and uncovered a critical logic flaw that allows for total transaction bypass."
tags: ["Web Vulnerabilities", "Logic Flaws", "Phishing", "Fintech"]
categories: ["Security Research"]
featured_image: "featured.png"
---

## Executive Summary
In Tel Aviv, **Linx** has become a dominant payment and loyalty platform, integrated into over 100 restaurants and bars. While the application provides a seamless user experience, the underlying trust model relies on a dangerous assumption: that the human verifier (the waiter) is a reliable validator of a digital state.

I accepted a challenge to audit the system's integrity. My research uncovered a **Critical Business Logic Vulnerability** that allows an attacker to bypass the entire payment flow, effectively "settling" bills without any actual transfer of funds or credit points.

{{< alert icon="shield" cardColor="#ff48006b" iconColor="#f1faee" >}}
**Vulnerability Risk Assessment: HIGH**
* **Vector:** Business Logic / Social Engineering Hybrid
* **Impact:** High (Direct financial loss to merchants and platform)
* **Exploitability:** High (Unauthenticated; requires no specialized hardware)
* **Complexity:** Medium (Requires high-fidelity UI replication and control of a look-alike domain)
* **Remediation Status:** **Not Patched**
{{< /alert >}}

---

## System Architecture & Trust Assumptions

To understand the vulnerability, we must examine the standard "Happy Path" transaction flow:

1. **Initiation:** The user enters the bill amount and desired credit (Linx points) usage into the mobile app.
2. **Challenge Generation:** The app generates a QR code on the user's device.
3. **Verification:** The waiter scans the QR code, which directs their device to `biz.linx.win`.
4. **Manual Accounting:** The waiter is instructed by the web interface to manually credit the bill in the restaurant's POS (Point of Sale) system.
{{< details summary="Technical Context: The 'Be'hakafa' (Settlement) Model" >}}
In the Israeli hospitality sector, Linx operates on a "settlement" model. When a waiter confirms the transaction, they mark the bill in their POS system as **"Charged to Linx (Be'hakafa - בהקפה)."** This creates a ledger entry stating that the restaurant is now owed money by the Linx platform. Because the verification is manual and happens on the user's device, the restaurant provides the service on credit, only to discover during monthly reconciliation that the transaction never existed on the Linx backend.
{{< /details >}}

5. **Finalization:** The waiter clicks "Confirm" on the web interface, and the user's screen displays a success message.

---

## Root Cause: Business Logic Flaw

The fundamental security failure in this ecosystem is a business logic flaw. 

In secure payment architectures (such as EMV or modern 3DS flows), the **Merchant** generates a challenge that the **User** must satisfy. In the Linx model, the **User** provides the "source of truth" (the QR code) to the Merchant.

By allowing the customer to provide the entry point, the application cedes control of the **execution environment** to a potential attacker. As an attacker, I do not just provide a code; I provide the entire browser session the waiter uses to verify the payment.



---

## Vulnerability Analysis: The Phishing-Logic Hybrid

This exploit represents a hybrid attack that bridges digital flaws with human-centric social engineering:

1. **Unauthenticated Entry Point:** The waiter scans a code from an untrusted device without a prior cryptographic handshake between the restaurant's POS and the Linx server.
2. **Lack of Out-of-Band Verification:** There is no independent confirmation (e.g., a Push notification to a merchant tablet). The restaurant's only "proof" of payment is pixels on a screen they do not control.
3. **Cognitive Bias Exploitation:** The "Success" UI triggers a manual accounting action. If the interface looks authentic and the domain appears legitimate, the human verifier assumes the financial transaction is already settled on the backend.

---

## Exploit Development: Ecosystem Replication

To demonstrate the risk, I developed a high-fidelity PoC consisting of two components:

### 1. The Generator (Attacker Interface)
A React-based dashboard that mimics the Linx app. It allows the attacker to input the restaurant name and bill totals, generating a QR code that points to a rogue domain.
![Comparison: Side-by-side comparison of the legitimate Linx mobile app interface and a custom-built rogue QR code generator mimicking its design.](img/figure1.png "*Figure 1: Side-by-side comparison: The authentic Linx mobile UI (left) vs. the rogue QR code generator (right) used in the exploit.*")


### 2. The Phisher (Rogue Verification Interface)
I registered `linx.ovh` and configured the subdomain `biz.linx.ovh` to mirror the official `biz.linx.win`. Using a combination of React and PHP, I recreated the exact CSS, haptics, and workflow of the official confirmation page.

![An animated GIF alternating between the official Linx merchant verification screen and a spoofed version, showing identical layouts and animations.](img/figure2.gif "*Figure 2: UX Fidelity: The spoofed verification interface (right) replicates the official branding and animations, making the deception nearly impossible to detect during a busy shift.*")

To maximize the success of the deception, I focused on High-Fidelity UX Replication. Interestingly, the rogue interface actually outperformed the original in terms of responsiveness and animation fluidity. By providing a 'slicker' and more modern UI/UX, I leveraged the psychological bias where users equate visual polish with system legitimacy, effectively neutralizing any potential suspicion from the waiter during a high-speed shift.

---

## Technical Implementation & OpSec

To ensure the exploit was resilient against manual inspection, I implemented several Operational Security (OpSec) measures.

### State Management & Evidence Sanitization
Initially, passing transaction data via Base64 in the URL created a fingerprinting risk. I moved to a **UUID-based lookup system**.

* **The "Vanishing" Act:** To prevent forensic analysis, the UUID is extracted from the URL and moved to `sessionStorage` immediately upon page load.
{{< details summary="Why sessionStorage?" >}}
I chose `sessionStorage` over `localStorage` or `Cookies` to create a **self-destruct mechanism**. 

Because `sessionStorage` is non-persistent and scoped to the tab, the data is automatically purged the moment the waiter closes the tab. This ensures that the "evidence" only exists for the duration required to deceive the human in the loop. 
{{< /details >}}

```javascript
// Prevent easy sharing/inspection by clearing URL state
useEffect(() => {
  const uuidRegex = /[0-9a-f\-]{36}/i;
  const transactionId = window.location.pathname.match(uuidRegex)?.[0];
  if (transactionId) {
    // Persist ID in volatile memory for the current session
    sessionStorage.setItem('transaction-id', transactionId);
    // Scrub UUID from address bar
    window.history.replaceState(null, '', '/approve_transaction/');
    // Validate against the rogue backend
    validateTransaction(transactionId);
  }
}, []);
```

### Forensic Considerations: The History Gap
While `replaceState` cleans the address bar, the initial request persists in the browser's global history logs. To mitigate this, I implemented a strict **Server-Side TTL (Time-To-Live)**. While I used a 10-minute window for this demonstration, the attacker can calibrate this window to ensure the link expires shortly after the anticipated verification, leaving no 'live' evidence for later forensic retrieval. The backend invalidates the UUID after 10 minutes, ensuring that even if the link is recovered from history later, it leads to a 404 "Dead End."

```php
// Server-Side Session Invalidation (PHP)
$createdAt = $transaction['created_at'];
if (time() - $createdAt > 600) { // Configurable validity window
    http_response_code(404);
    exit; 
}
```

### Environment Masking via `.htaccess`
I utilized Apache's mod_rewrite to ensure that API endpoints and file paths were identical to the target's infrastructure. This ensures that even if the "Network" tab is opened, the XHR requests appear to hit legitimate-looking endpoints.

```apache
RewriteEngine On
# Mirroring the official transaction path
RewriteRule ^approve_transaction/([a-fA-F0-9\-]{36})$ index.php?id=$1 [L,QSA]
# Mimicking background API calls
RewriteRule ^api/transaction/approve/?$ api/complete_transaction.php [L,NC]
```

### Defensive Recommendations
To bridge the "Physical-Digital Gap," the confirmation logic must be decoupled from the user's device:

1. **Synchronous Server-to-Merchant Verification:** The merchant's POS or a dedicated tablet should receive a direct WebSocket or Push notification from the Linx server once a payment is authorized.

2. **Cryptographic Binding:** QR codes should contain a short-lived, signed JWT (JSON Web Token) that the merchant device validates against a public key.

3. **Mutual Authentication:** The merchant interface should be behind a dedicated authentication layer, not a public-facing web app.

4. **Out-of-Band Validation (OTP):** Implementing a 5-digit OTP (similar to Cibus/10Bis) provides a simple, low-cost secondary factor that validates the transaction state.

---

## Conclusion

This research serves as a case study in how security is only as robust as its weakest logical link. By analyzing the intersection of business logic and the high-pressure environment of a hospitality worker, I was able to bypass the entire financial utility of the platform. It is a reminder that in the "Physical-Digital Gap," human intuition is often exploited to cover for missing technical verifications.

![A physical paper receipt from a restaurant showing a successful transaction processed as 'Linx', proving the exploit successfully bypassed the payment backend.](img/figure4.jpg?w=300 "*Figure 4: Proof of Concept: A physical receipt obtained during field testing, confirming the restaurant's POS system accepted the spoofed transaction as a valid payment.*")

---

## 🧪 Live Proof of Concept (PoC)

To demonstrate the mechanics of the attack without enabling real-world abuse, I deployed a non-production simulation that intentionally prevents successful settlement.

The proof-of-concept includes a mandatory, non-bypassable security warning displayed before any transaction state is rendered. This warning explicitly identifies the interface as a security simulation and prevents the completion of any real payment flow.

As a result, the PoC demonstrates the exploit’s mechanics while materially reducing the risk of real-world financial loss or unauthorized misuse.

> **[🔗 Launch Live Demo](https://biz.linx.ovh/init_transaction/)**

**Technical focus of this PoC:**
* **UX/UI Fidelity:** Observe the high-fidelity replication of the user interface.
* **Session Management:** Watch the address bar to see the `replaceState` logic clear the UUID, while `sessionStorage` maintains the transaction context.
* **Backend TTL:** Attempting to access an expired session will trigger a 404 response.

---

## 🛡️ Responsible Disclosure Timeline

* **January 3, 2026:** Discovery of the business logic flaw.
* **January 8, 2026:** Vulnerability verified in a live environment. Linx team notified.
* **January 12, 2026:** Full technical report submitted to the Linx security/engineering team. Acknowledgement from the team; vulnerability confirmed.

*Note: All research was conducted ethically. No businesses were harmed; any funds used during live testing were returned to Linx.*

---

### Researcher's Perspective
As a Software Developer, I approach security through the lens of **system logic** and **architectural assumptions**. This vulnerability highlights that "breaking" a system doesn't always require a code-level bug; sometimes, it only requires breaking the assumption of trust between the digital and physical worlds.

If you’d like to discuss this research or related security topics, you can reach me on [LinkedIn](https://www.linkedin.com/in/rony-utevsky/).