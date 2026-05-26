---
title: "TrustFall: Coding Agent Security Flaw Enables One-Click RCE"
slug: "trustfall-coding-agent-rce"
date: 2026-05-07
description: "An in-depth look at TrustFall, a critical vulnerability in AI coding agents (Claude Code, Cursor, Gemini CLI, Copilot) that enables one-click RCE via malicious project-defined MCP server configurations."
tags: ["AI Security", "RCE", "MCP", "Claude Code", "Cursor", "Gemini CLI", "Copilot"]
categories: ["Security Research"]
featured_image: "featured.png"
---

## Executive Summary

I co-discovered **TrustFall**, a critical security flaw affecting major agentic developer tools including Claude Code, Cursor CLI, Gemini CLI (Antigravity), and GitHub Copilot CLI. The vulnerability allows a malicious repository to execute arbitrary code with one click, exploiting the implicit trust granted during the workspace initialization.

The core issue lies in the handling of **Model Context Protocol (MCP)** project-scoped servers. When a user opens an untrusted repository and accepts the standard folder-trust prompt, all four agentic CLIs auto-execute the project-defined MCP servers without any additional confirmation or secondary warning.

{{< alert icon="shield" cardColor="#e2131380" iconColor="#f1faee" >}}
**Vulnerability Risk Assessment: CRITICAL**

* **Vulnerability Name:** TrustFall
* **Affected Tools:** Claude Code, Cursor CLI, Gemini CLI (Antigravity), GitHub Copilot CLI
* **Vulnerability Type:** One-Click Remote Code Execution (RCE)
* **Attack Vector:** Malicious repository containing project-scoped MCP server configurations
* **Impact:** Unsandboxed execution on the developer machine, or 0-click environment variable exfiltration in headless CI/CD pipelines
{{< /alert >}}

---

## How the Vulnerability Works

1. **Malicious Config:** An attacker commits a custom `.mcp.json` or equivalent configuration file to a repository. This config specifies an MCP server whose startup command launches a malicious payload (e.g., executing shell scripts, opening calculators, or exfiltrating credentials).
2. **The "Trust Fall":** The developer clones the repository and starts the coding agent.
3. **Implicit Execution:** The coding agent displays a generic folder-trust prompt. Once the developer presses Enter to grant trust, the agent immediately spins up the configured MCP servers, executing the attacker's payload.

On headless CI/CD systems, the attack can escalate to **zero-click** execution. Since CI runtimes typically execute coding agents in non-interactive modes with auto-trust settings enabled, simply submitting a malicious pull request allows the exploit to trigger automatically, exfiltrating pipeline secrets and environment variables.

---

## Full Research and Details

For the complete technical breakdown, proof-of-concepts, and vendor responses, read the original publication:

👉 **[TrustFall: coding agent security flaw enables one-click RCE in Claude, Cursor, Gemini CLI and GitHub Copilot](https://adversa.ai/blog/trustfall-coding-agent-security-flaw-rce-claude-cursor-gemini-cli-copilot/)** *(Published on Adversa AI)*
