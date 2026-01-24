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

Zero-day DOM XSS in `react-show-more-text` (30K+ weekly downloads): unsafe layout measurement logic bypasses React's escaping, achieving zero-click JavaScript execution when content fits in one line. This represents a new vulnerability class — **layout-dependent execution paths** in trusted UI libraries. Widely deployed in employee directories and user profiles, making it a supply-chain wormable primitive.

**[Read the full technical analysis →]({{< relref "/posts/react-show-more-text">}})** | **[See corporate worm exploitation →]({{< relref "/posts/xss-worm">}})**
