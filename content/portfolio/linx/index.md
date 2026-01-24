---
title: "The Trust Fall: Bypassing a City-Wide Payment Ecosystem"
slug: "linx-payment-bypass"
date: 2026-01-12
description: "A critical business logic vulnerability in a fintech payment platform where merchants treat a user-controlled screen as proof of payment."
tags: ["Business Logic", "Fintech", "Trust Boundary"]
categories: ["Security Research"]
showReadingTime: false
thumbnail: "featured.png"
---

A city-wide payment platform used across 100+ hospitality venues trusted customer-controlled screens as proof of payment. Merchants rely on a visual “success” state and settle bills immediately via POS, but there is no cryptographic binding or backend confirmation. This trust-boundary failure enables transaction bypass: service is delivered in real time, while missing funds are only discovered during later reconciliation — creating silent, scalable financial exposure.

**[Read the full technical analysis →]({{< relref "/posts/linx">}})**
