---
title: "Exploiting Layout Logic for DOM-Based XSS in react-show-more-text"
slug: "dom-xss-react-show-more-text"
date: 2026-01-20
description: "A zero-day DOM XSS vulnerability in a popular React UI component caused by unsafe layout measurement and HTML restoration logic."
tags: ["Zero-Day", "React", "XSS", "Supply Chain"]
categories: ["Security Research"]
showReadingTime: false
thumbnail: "featured.png"
---

**Impact:** Zero-click JS execution | **Reach:** 30k+ Weekly Installs |  
**Severity:** High | CVSS 8.1 (v3.1)

Identified a novel vulnerability class: layout-dependent execution paths. By exploiting how this component measures container dimensions, I bypassed React’s built-in XSS protections.
* **The Flaw:** Unsafe interaction between layout-measurement logic and HTML restoration.
* **The Trigger:** JavaScript executes automatically when attacker-controlled strings fit entirely within the component’s pixel boundaries.
* **The Sink:** Bypasses React's escaping model and triggers arbitrary execution through `dangerouslySetInnerHTML`.
* **The Exploit:** Chained into a **wormable supply-chain primitive** within a large corporate environment.
* **Status**: Authored the official security patch (v1.7.2).

**[Read Technical Deep-Dive →]({{< relref "/research/react-show-more-text">}})**  
**[See Corporate Worm Exploitation →]({{< relref "/research/xss-worm">}})**
