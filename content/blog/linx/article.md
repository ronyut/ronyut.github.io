---
title: "The Trust Fall: Bypassing a City-Wide Payment Ecosystem"
date: 2026-01-10
description: "How a logical flaw in a popular payment app allowed for 'free' meals via technical phishing and UI replication."
tags: ["Web Vulnerabilities", "Logic Flaws", "Phishing", "Fintech"]
categories: ["Security Research"]
---

## The Hook
In Tel Aviv, an app called **Linx** has become a staple for the local youth, offering cashback and credit points at over 100 restaurants and bars. While the UI is slick, the underlying trust model relies on a dangerous assumption: that the human element (the waiter) is a reliable validator of a digital state.

I accepted a challenge to see if the system could be bypassed. What I found wasn't a memory leak or a SQL injection, but a massive **Logical Loophole** that allows anyone to "pay" without spending a single shekel.

---

## The Architecture: A Chain of Trust

To understand the flaw, we must look at the standard transaction flow:

1. **User** enters, in the app, the bill amount and how many credit points to use.
2. **App** generates a QR code.
3. **Waiter** scans the QR code, which opens `biz.linx.win`.
4. **Waiter** sees instructions to manually charge the "Linx" button on the POS (Point of Sale) terminal.
{{< details summary="Technical Note: \"Be'hakafa\" (Credit-based Settlement)" >}}
> In the Israeli hospitality sector, Linx operates on a "settlement" model. When a 
> waiter confirms the transaction, they mark the bill in their POS system as 
> **"Charged to Linx (Be'hakafa - בהקפה)."** This creates a ledger entry stating that 
> the restaurant is owed money by the Linx platform. Because the verification 
> is manual and happens on the user's device, the restaurant provides the 
> service on credit, only to discover much later—during monthly reconciliation—that 
> the transaction never actually existed on the Linx backend.
{{< /details >}}
5. **Waiter** confirms the transaction on the website.

## The Flaw: Inversion of Trust
The fundamental security failure of the Linx ecosystem is an **Inversion of the Trust Model**. 

In most secure payment architectures (like EMV or modern POS systems), the **Merchant** generates a challenge that the **User** must satisfy. In the Linx model, the **User** provides the "source of truth" (the QR code) to the Merchant.

By allowing the customer to provide the QR code, the app cedes control of the **execution environment**. As an attacker, I don't just provide a code; I provide the entire web browser session that the waiter will use to "verify" the payment.

---

## Vulnerability Analysis: The Phishing-Logic Hybrid

This isn't a simple phishing attack; it's a **Logical Breach** of the business flow.

1.  **Unauthenticated Entry Point:** The waiter scans a code from a random device without any prior handshake between the restaurant's POS and the Linx server.
2.  **Lack of Out-of-Band Verification:** There is no "Push" notification or independent confirmation. The restaurant's only "proof" is the pixels on a screen that they don't own.
3.  **Human-in-the-Loop Exploitation:** The "Success" screen triggers a manual accounting action (**"Be'hakafa"**). The software exploits the waiter’s cognitive bias—if the screen looks right and the URL says "Linx," the waiter assumes the financial transaction is already settled.

---

## The Attack: Replicating the Ecosystem

I built a two-part system to exploit this trust.

### 1. The "Generator" (Attacker UI)
I developed a React-based form to simulate the Linx app. It allows the attacker to input the restaurant name, total bill, and desired "fake" credit. Upon submission, it generates a QR code pointing to my rogue domain.

### 2. The "Phisher" (Rogue Confirmation Page)
I registered `linx.ovh` and set up the subdomain `biz.linx.ovh`. Using PHP and React, I recreated the exact CSS and behavior of the official confirmation page.

> **[INSERT SCREENSHOT HERE: Side-by-side comparison of the real biz.linx.win and your biz.linx.ovh]**
> *Caption: Spot the difference: Official vs. Rogue.*

---

## Technical Implementation & OpSec

As a developer, I wanted to ensure the "exploit" was robust and resistant to casual inspection.

### UUID & State Management
Initially, I passed transaction data via Base64 in the URL. I realized this was a fingerprinting risk. I pivoted to a **UUID-based lookup system**:

* The QR contains a unique ID: `biz.linx.ovh/approve_transaction/?id=550e8400-e29b...`
* The backend (PHP) validates the UUID. 
* **The "Vanishing" Act:** To prevent forensic analysis, the UUID is wiped from the address bar immediately upon page load and moved to `sessionStorage`.
{{< details summary="Why I chose `sessionStorage`" >}}
A key part of the exploit was ensuring that no trace of the "fake" transaction remained on the waiter's device after the interaction. I specifically chose `sessionStorage` over `localStorage` or `Cookies` because it's the only way to make sure that the data is automatically wiped as soon as the tab is closed. This provides a "self-destruct" mechanism: once the waiter sees the "Success" screen and closes their browser, the UUID and transaction data disappear.

This implementation ensures that the "evidence" only exists for the few seconds required to deceive the human in the loop.

Anyway, the UUID will still be accessible via browser history log (I found no way to conceal it).
{{< /details >}}

```javascript
// React logic to hide the UUID from the address bar immediately
useEffect(() => {
  const params = new URLSearchParams(window.location.search);
  const transactionId = params.get('id');

  if (transactionId) {
    // 1. Store the ID for the session so the app can still use it
    sessionStorage.setItem('transaction-id', transactionId);

    // 2. Clean the URL immediately so the ID can't be easily copied/shared
    window.history.replaceState(null, '', window.location.pathname);
    
    // 3. Fetch faked transaction details via PHP API
    validateTransaction(transactionId);
  }
}, []);
```

I should note that While `sessionStorage` and `replaceState` provide excellent "surface-level" stealth, they do not bypass deep forensics:
* **The History Gap:** Modern browsers log the initial request URL before JavaScript can intervene. A dedicated audit of the browser's history database would reveal the UUID.
* **The Solution:** To mitigate this, I implemented a strict **Server-Side TTL**. Even if the UUID is recovered from the history log 10 minutes later, the backend has already purged the record, rendering the link a "Dead End."

### Time-To-Live (TTL)
To ensure no "ghost" transactions stayed active, I implemented a TTL. The link is only valid for a window of a few minutes. If scanned later, the PHP backend returns a 404, mimicking a standard session timeout.
```php
// Simple TTL Logic in PHP
$createdAt = $transaction['created_at'];
if (time() - $createdAt > 600) { // 10 minute window
    http_response_code(404);
    exit;
}
```

### Backend Mocking  via `.htaccess`
I used Apache's `mod_rewrite` to ensure the internal API calls and page paths matched the original Linx structure exactly. This ensures that even if the "Network" tab is opened, the XHR requests appear to hit legitimate-looking endpoints.

```apache
RewriteEngine On
# Mimicking the official path structure for credibility
RewriteRule ^approve_transaction/([a-fA-F0-9\-]{36})$ index.php?id=$1 [L,QSA]
# Mimicking API calls
RewriteRule ^api/transaction/approve/?$ api/complete_transaction.php [L,NC]
```

### The Developer's Remediation
This is a classic "Physical-Digital Gap" problem. To secure this, the "Confirmation" must move away from the user's screen:

1. Synchronous Verification: The waiter's POS should receive a notification directly from the Linx server once the user clicks "Pay."

2. Digital Signatures: If QR codes are used, they must contain a short-lived JWT (JSON Web Token) signed by the server that the waiter's device must verify.

3. Mutual Auth: The merchant's interface should not be a public-facing website accessible by any browser; it should require merchant-specific authentication.

4. Use an OTP, like in Cibus/10Bis, where the customer reads a sequence of 5 digits that validate the transaction.

---

## Conclusion
This project is a reminder that security is only as strong as its weakest logical link. By understanding the business logic and the human behavior of a busy bartender, I was able to bypass the entire financial utility of the app.

---

## 🧪 Live Proof of Concept (PoC)

Since Linx has officially patched this logical vulnerability, I have made a sanitized version of the research environment available for review.

> **[🔗 Launch Live Demo](https://biz.linx.ovh/init_transaction/)**

**Technical focus of this PoC:**
* **UX/UI Fidelity:** Observe the CSS-based replication of the "Success" state.
* **Session Management:** Watch the address bar to see the `replaceState` logic clear the UUID while `sessionStorage` maintains the transaction context.
* **Backend TTL:** The demo simulates the server-side expiration logic that renders the history log entries useless after a set duration.

---

## 🛡️ Responsible Disclosure Timeline

* **January 3, 2026:** Initial discovery of the logical flaw.
* **January 8, 2026:** Vulnerabiliy tested IRL and confirmed. Notified the Linx team.
* **Date + x Days:** Full technical report submitted to the Linx security/engineering team.
* **Date + x Days:** Acknowledgement from the team; vulnerability confirmed.
* **Date + x Days:** **Patch Deployed.** The trust model was updated to include [mention the fix, e.g., Backend-to-POS confirmation].
* **Today:** Public disclosure of findings.

---

### Note
All findings were reported to the Linx team prior to this publication. Apart from one business wherin I tested the vulnerability (and returned the money afterwards), no businesses were harmed during this research.

---

### Researcher's Perspective
As a Software Developer, I tend to look at security through the lens of **system logic** and **user flow**. This vulnerability was particularly interesting because it didn't require "breaking" code in the traditional sense; it required breaking the **assumption of trust** between the digital UI and the physical world. 

Identifying these architectural gaps is where I thrive. If you are looking for a researcher who understands how to build—and how to think like those who build—I'd love to chat.

**Looking for a research position? [Let's connect on LinkedIn](your-profile-url).**