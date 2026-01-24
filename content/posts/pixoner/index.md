---
title: "Breaking the Paywall: How a 'Lazy' Search Implementation Compromised a Paywall"
slug: "pixoner-paywall-breach"
date: 2025-03-23
description: "Turning a search 'feature' into a mass-disclosure exploit: How partial UUID matching allowed for the automated scraping of private tourist photo albums."
tags: ["Broken Access Control", "Business Logic", "Web Security", "Red Teaming", "Brute Force"]
categories: ["Security Research"]
featured_image: "featured.png"
---

## Executive Summary

While a friend was enjoying a scenic icebreaker cruise in Lapland, I was presented with a different kind of challenge. The tour used a QR-code tagging system to track tourists and sell photo albums at a premium. Access to these albums was guarded by a unique UUID, entered into a single search field on the vendor's website.

By investigating the search behavior, I discovered a critical design flaw: the system accepted **partial substrings** of UUIDs and automatically redirected users to the first matching record. By chaining this "shortcut" behavior with an optimized recursive brute-force script, I demonstrated how an attacker could systematically discover and access every private album in the database without payment.

## The Challenge: A Digital Paywall

The system was straightforward:
1. Tourists wear a QR code on their chest during the tour.
2. Photographers take hundreds of shots of the participants.
3. To access the photos, you must pay a fee to receive a unique 36-character UUID (e.g., `af512aa6-64ce-4a2b-a0df-0b758cb29e3a`).
4. Entering this UUID on the website grants access to the high-resolution downloads.

My friend, skeptical of the high price, challenged me to find a way in. I accepted.

## The Discovery: The "Shortcut" Vulnerability

While testing the search input, I noticed an interesting "feature." Instead of requiring a full 36-character string, the backend was performing a `startsWith()` query. 

If I entered a single character, the server would look for any UUID starting with that character and immediately redirect me to the first result it found.
* **Input:** `a` → **Redirects to:** `af512aa6-...`
* **Input:** `aa` → **Redirects to:** `aa5cfaea-...`

This behavior effectively turned a "needle in a haystack" problem (guessing a 128-bit UUID) into a **searchable tree traversal problem.**



## The Exploit: Optimized Brute-Force

### 1. Mapping the Search Space
Since UUIDs are hexadecimal (0-9, a-f), each character added to the search string narrows the results. I calculated that testing all combinations up to 5 characters long would be sufficient to map the active database.

The total search space for a 5-character depth is:
$$16^1 + 16^2 + 16^3 + 16^4 + 16^5 =\newline\newline1,118,480\text{ combinations}$$

### 2. Pruning the Tree
The most critical realization was that the search space is **sparse**. If a prefix like `b42` returns no results, there is no reason to test `b420`, `b421`, or any further children. 

By implementing a recursive check, the script only "dives deeper" into prefixes that are confirmed to exist.

### 3. Concurrency and Stealth
Executing 1.1 million requests sequentially would be incredibly slow due to network latency. To solve this, I used Python's `threading` module to run 100 simultaneous workers.

**Why 100 threads?**
I found that 100 threads are the sweet spot that allowed me to "saturate" the connection. While one thread is waiting for a response from the server in Europe, others are already sending the next batch of keys. However, this comes with a risk: sending too many requests from a single IP can trigger a WAF (Web Application Firewall) or a DoS (Denial of Service) protection block.

In this specific instance, the target's lack of aggressive rate-limiting allowed the script to complete. In a more hardened environment, a researcher would need to implement **proxy rotation** and **request jitter** to mimic organic human traffic.

### 3. The Implementation
The core logic uses a worker pattern to manage the search queue. By using a session object, we maintain persistence and handle the necessary CSRF tokens.

```python
import threading
import requests

# Worker function to process the search queue
def worker():
    while not queue.empty():
        prefix = queue.get()
        
        # We use a session to maintain cookies and CSRF tokens
        response = session.post(TARGET_URL, data={
            "DownloadKey": prefix,
            "EventCode": "J33ZWY",
            "__RequestVerificationToken": token
        })

        # Detection logic: Check for the 302 Redirect to a full UUID
        if response.history and response.history[0].status_code == 302:
            found_url = response.url
            with lock:
                if found_url not in MATCHES:
                    MATCHES.add(found_url)
                    print(f"[+] Found Album: {found_url}")
                    # Recursive logic: explore the next hex depth
                    add_next_level_to_queue(prefix)
        
        queue.task_done()

# Main execution loop: Launching the 100-worker thread pool
def run_exploit():
    for i in range(100):
        t = threading.Thread(target=worker)
        t.daemon = True
        t.start()
    queue.join()
```


## Results
The script successfully extracted 74 unique album UUIDs from the cruise date. Each UUID belonged to a different passenger. While I suggested using AI face recognition to find my friend's specific photos, he opted to manually review the 74 albums.

## Impact & Ethics
This vulnerability allowed for unauthorized mass-disclosure of private data. In a real-world scenario, this would enable:
* **Privacy Violations**: Scraping thousands of private images.
* **Revenue Loss**: Bypass of the paywall

Testing was limited to demonstrating the existence of the vulnerability.
No images were downloaded or stored beyond confirming unauthorized access paths.

## Conclusion & Remediation
The "shortcut" intended to make the site user-friendly became the tool used to dismantle its security.  
The fix is simple: Access keys must require an exact match.  
Recommendations:  
* Disable partial matching in database queries.
* Enforce Rate Limiting based on IP and session tokens.
* Require secondary authentication factor (e.g., an email address) to view sensitive content.

Looking to secure your own data-access layers or conduct a deep-dive security audit? [Let's connect on LinkedIn](https://www.linkedin.com/in/rony-utevsky/).**

<!-- https://my.pixoner.com/en/Download/AllPhotos/J33ZWY -->