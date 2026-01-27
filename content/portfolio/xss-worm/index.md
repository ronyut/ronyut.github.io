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

Chained an unpatched React library zero-day with backend validation flaws to create a **self-propagating stored XSS worm** in a corporate Employee Directory. The OOO field accepted raw HTML, triggering XSS via `react-show-more-text` on trusted origin. Weaponized with **WebSocket C2 for persistent control** and autonomous worm propagation, enabling credential harvesting (confirmed in live testing), lateral movement to expense/ESPP/Travel systems, and **post-employment attack survivability**. Perfect storm: untrusted input + supply-chain vulnerability + trusted domain + persistent infrastructure.

**[Read the full technical analysis →]({{< relref "/research/xss-worm">}})** | **[See the underlying zero-day →]({{< relref "/research/react-show-more-text">}})**
