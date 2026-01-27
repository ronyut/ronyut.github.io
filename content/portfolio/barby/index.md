---
title: "The 1-Shekel Ticket: Broken Access Control, Mass PII Exposure, and Price Manipulation in a Concert Ticketing Platform"
slug: "barby-ticketing-idor-price-manipulation"
date: 2024-09-17
description: "Systematic access control failures in a major ticketing platform enabling mass PII exposure, account takeover, and arbitrary price manipulation."
tags: ["IDOR", "Broken Access Control", "Business Logic", "PII Exposure"]
categories: ["Security Research"]
showReadingTime: false
thumbnail: "featured.png"
---

A major Israeli ticketing platform exposed systemic access control failures: sequential IDs leaked ~100K customer records (PII, partial credit cards), client-side OTP generation enabled instant account takeover, and client-controlled pricing allowed purchasing valid concert tickets for 1 ILS. This wasn't one bug — it was a complete trust boundary collapse across authentication, authorization, and payment flows.

**[Read the full technical analysis →]({{< relref "/research/barby">}})**
