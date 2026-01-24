---
title: "Breaking the Paywall: How a 'Lazy' Search Implementation Compromised a Paywall"
slug: "pixoner-paywall-breach"
date: 2025-03-23
description: "Turning a search 'feature' into a mass-disclosure exploit via partial UUID matching and optimized brute-force enumeration."
tags: ["Broken Access Control", "Business Logic", "Web Security", "Brute Force"]
categories: ["Security Research"]
showReadingTime: false
thumbnail: "featured.png"
---

A tourist photo vendor used unique UUIDs to protect paid albums. I discovered the search field accepted **partial UUID matches** and auto-redirected to the first result. By exploiting this "convenience feature" with a recursive brute-force script, I enumerated 74 private albums in under 30 minutes — achieving complete paywall bypass without payment.

**[Read the full technical analysis →]({{< relref "/posts/pixoner">}})**
