---
title: "Reverse Engineering a 3D Unity Web App to Win a Chocolate Contest"
slug: "kinder-challenge-unity-reverse-engineering"
date: 2024-07-21
description: "Dismantling a compiled Unity bundle, analyzing 3D mesh data, and automating UI input to solve a 'guess the number' challenge with 100% accuracy."
tags: ["Reverse Engineering", "Unity", "UI Automation", "Game Hacking"]
categories: ["Security Research"]
showReadingTime: false
thumbnail: "featured.jpg"
---

Kinder Israel ran a contest: guess the number of chocolate bars in a 3D suitcase. Instead of guessing, I decompiled the Unity WebAssembly bundle, extracted the 3D scene, and used ProBuilder's vertex clustering to count individual objects hidden within a merged mesh. An AutoHotkey script automated submission. Result: 100% accuracy. Key lesson: **client-side abstraction is not a security boundary**.

**[Read the full technical analysis →]({{< relref "/posts/kinder">}})**
