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

**Impact:** Paywall Bypass | **Type:** Broken Access Control

A tourist photo vendor used unique UUIDs to protect paid albums. I turned a UX "convenience feature" into a mass-disclosure primitive by exploiting the vendor's use of partial UUID matching. This transformed a 128-bit "needle in a haystack" problem into a searchable tree traversal.
* **The Flaw:** The search field used a `startsWith()` query logic. Entering a partial album key (even a single character) triggered an automatic redirect to the first matching private album.
* **The Exploit:** Developed a recursive brute-force script in Python that "pruned" the search space: if a prefix returned no results, the script skipped all its children, drastically reducing the required requests.
* **The Result:** Used concurrent threads to map the sparse search space, successfully extracting relevant unique private albums in a few hours.
* **The Takeaway:** UX shortcuts should never cross security boundaries; obfuscation via UUID is not a substitute for proper authorization.

**[Read Full Technical Analysis →]({{< relref "/fun-hacking/pixoner">}})**
