---
title: "SymJack: the approval prompt is lying to you. A symlink-hijack RCE in five AI coding agents"
slug: "symjack-approval-prompt-lying-symlink-rce"
date: 2026-05-26
description: "A critical security bypass in six leading AI coding agents (Claude Code, Cursor, Antigravity, Copilot, Grok, Codex) where deceptive copy/write commands chain with workspace symlinks to achieve RCE."
tags: ["AI Security", "SymJack", "RCE", "Symlinks", "MCP", "Claude Code", "Cursor", "Antigravity", "Copilot", "Grok", "Codex"]
categories: ["Security Research"]
featured_image: "featured.png"
---

## Executive Summary

Following up on the initial **TrustFall** discovery, I identified a critical security bypass affecting six prominent AI coding agents: Claude Code, Cursor CLI, Antigravity (Gemini CLI), GitHub Copilot CLI, Grok Build, and Codex CLI. This bypass allows a weaponized repository to trick the user into granting Remote Code Execution (RCE) via a deceptively innocent tool-approval prompt.

The attack leverages symbolic links (symlinks) committed directly to a repository to defeat the agents' built-in sensitive-file warnings and file-system sandbox restrictions.

{{< alert icon="shield" cardColor="#ff48006b" iconColor="#f1faee" >}}
**Vulnerability Risk Assessment: HIGH**

* **Vulnerability Name:** Symlink Trust Bypass (The Lying Approval Prompt)
* **Affected Tools:** Claude Code, Cursor CLI, Antigravity, GitHub Copilot CLI, Grok Build, Codex CLI
* **Vulnerability Type:** Trust Bypass / Arbitrary File Overwrite leading to RCE
* **Attack Surface:** Any developer environment or automated CI/CD pipeline using AI coding agents on untrusted repositories
{{< /alert >}}

---

## How the Vulnerability Works

1. **Weaponized Symlinks:** The attacker commits symlinks in the workspace that mimic harmless files but point directly to the agent's sensitive configuration directories (e.g., `.mcp.json` or agent settings).
2. **Deceptive Prompts:** The agent is prompted by instructions in the repo to perform a standard file copy or write operation (e.g., copying a video file from `media/vid0.mp4` to a symlinked path like `docs/vid0.mp4` that resolves to `.mcp.json`).
3. **The Symlink Follow:** When the user reviews the prompt, the agent displays an innocent-looking copy command, keeping the user unaware of the real target. Once approved, the underlying kernel follows the symlink and overwrites the agent's config with the attacker's JSON payload disguised inside the copied file.
4. **Execution:** Upon next execution or agent restart, the overwritten configuration registers a malicious MCP server, resulting in unprompted command execution.

Similar to [TrustFall]({{< relref "/research/trustfall-coding-agent-rce" >}}), this technique is especially dangerous on headless CI/CD systems, where automated pull request pipelines instantly execute these operations without any human observation.

---

## Full Research and Details

For the complete technical breakdown, including proof-of-concept videos and detailed vendor responses, read the original publication:

👉 **[SymJack: the approval prompt is lying to you. A symlink-hijack RCE in six AI coding agents](https://adversa.ai/blog/the-approval-prompt-is-lying-to-you-symlink-rce-in-five-ai-coding-agents-claude-code-cursor-antigravity-copilot-grok-build/)** *(Published on Adversa AI)*
