---
name: opsec
description: >
  Operational security guardrails for cybersecurity work. Use this skill whenever
  the user shares indicators of compromise (IOCs), hashes, malware samples, IP addresses,
  domains, URLs, or any threat intelligence data. Also trigger when the user mentions
  PAP classification, asks about submitting or sharing data externally, or performs
  any analysis that could result in data leaving the local environment. Actively
  evaluate every proposed action against the PAP classification provided by the user.
---

# OpSec Skill

## Core Principle

The user determines PAP classification. Claude enforces it by evaluating every
proposed action before executing or suggesting it. When in doubt, ask before acting.

---

## PAP Protocol (Traffic Light Protocol for Permissible Actions)

PAP (Permissible Actions Protocol) defines what can be done with data, not just
who can receive it.

| Level | Meaning | Permitted Actions |
|---|---|---|
| **PAP:WHITE** | No restriction | Anything — public submission, sharing, publishing |
| **PAP:GREEN** | Community sharing | Share within trusted communities, no public posting |
| **PAP:AMBER** | Limited distribution | Internal use only, no external tools or services |
| **PAP:AMBER+STRICT** | Need-to-know only | Strictly local, no automated tools, no APIs |
| **PAP:RED** | No distribution | Local analysis only, nothing leaves the environment |

---

## Claude's Behavior

### 1. Track Classification
When the user provides a PAP classification for any data, retain it for the
entire conversation. Apply it to all subsequent actions involving that data.

### 2. Evaluate Before Acting
Before any action that could exfiltrate data, check against the classification:

**Actions that may exfiltrate data:**
- Submitting to VirusTotal, Any.run, Hybrid Analysis, or any public sandbox
- Querying external APIs (including Shodan, AbuseIPDB, etc.)
- Pushing to GitHub or any public/private remote repository
- Sending via email, Slack, or any communication tool
- Using web search with the data as query terms
- Calling any third-party service or MCP tool

### 3. Warn Proactively
If about to suggest or perform an action that violates the PAP level, stop and warn:

> ⚠️ **PAP:[LEVEL] — Action blocked**
> [Proposed action] would send this data to [destination].
> This violates PAP:[LEVEL] constraints.
> Suggest: [safe alternative if any]

### 4. Default Behavior (No Classification Given)
If the user shares IOCs, hashes, or samples without stating a PAP level:
- Treat as **PAP:GREEN** by default
- Inform the user: "No PAP classification provided — treating as PAP:GREEN. Confirm or override."

---

## Expanding to Malware Analysis Context

When used alongside malware analysis (e.g. with malwoverview):
- Determine PAP classification **before** running any analysis tool
- Do not pass sample hashes to external lookup services if PAP:AMBER or stricter
- Static analysis (local disassembly, string extraction, entropy analysis) is
  permitted at all PAP levels
- Dynamic analysis results (sandbox reports) may only be fetched if PAP:GREEN or WHITE

---

## Examples

**User:** "This hash is PAP:RED: `d41d8cd98f00b204e9800998ecf8427e`"
**Claude:** Retains classification. Will not query VirusTotal, will not include
in web searches, will not pass to any API.

**User:** "Look up this IP on Shodan"
**Claude (if IP is PAP:AMBER+STRICT):**
> ⚠️ PAP:AMBER+STRICT — Action blocked
> Querying Shodan would send this IP to an external service.
> Suggest: local WHOIS lookup or passive DNS if available offline.
