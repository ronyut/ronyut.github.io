---
title: "How an Unpatched Zero-Day in a React Library Exposed Corporate Data, Employee Credentials, and Financial Assets"
slug: "react-supply-chain-xss-worm"
date: 2025-12-28
description: "A React supply-chain zero-day that transformed an innocuous OOO field into a wormable corporate attack primitive."
tags: ["XSS", "Supply Chain", "Zero-Day", "Credential Harvesting", "Red Teaming"]
categories: ["Security Research"]
showReadingTime: false
thumbnail: "featured.png"
---

Chained an unpatched React library zero-day with backend validation flaws to create a **self-propagating stored XSS worm** in a corporate Employee Directory. The OOO field accepted raw HTML, triggering XSS via `react-show-more-text` on trusted origin. Result: autonomous worm propagation, contextual credential phishing (confirmed in live testing), and lateral movement to expense/ESPP systems. Perfect storm: untrusted input + supply-chain vuln + trusted domain.

**[Read the full technical analysis →]({{< relref "/posts/xss-worm">}})** | **[See the underlying zero-day →]({{< relref "/posts/react-show-more-text">}})**
