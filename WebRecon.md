# Bug Bounty Recon Journey — VDP Engagement
**Platform:** HackerOne  
**Program Type:** Vulnerability Disclosure Program (VDP)  
**Date:** May 2026  
**Tools:** Subfinder, Amass, Nmap, Burp Suite Community, proxychains, AnonSurf, dig, openssl, Python3

---

## Overview

This documents my first bug bounty recon engagement on a real-world VDP program. The goal was to passively enumerate the target's attack surface, identify security weaknesses, and attempt to produce a reportable finding. This writeup covers the methodology, tools, lessons learned, and outcome.

---

## Phase 1 — Subdomain Enumeration

### Tools Used
- `subfinder` — passive subdomain discovery
- `amass` — deeper passive enumeration
- Manual DNS resolution via `dig`

### Process
Started with passive subdomain enumeration on the primary domain. Combined results from multiple tools and deduplicated the output. Discovered several subdomains including:

- Main website
- Two VPN portals
- An agent/developer portal
- A travel portal
- A radius subdomain (no DNS record)

### Key Learning
Running through AnonSurf (Tor) caused issues with UDP-based DNS queries. Resolved by forcing TCP with `+tcp` flag and querying external resolvers directly (`@8.8.8.8`) rather than relying on the local stub resolver.

---

## Phase 2 — Infrastructure Fingerprinting

### Tools Used
- `dig` with TCP
- `openssl s_client`
- Python3 with custom `LegacyTLSAdapter`

### Process
Resolved each subdomain to identify hosting infrastructure. Noted a significant inconsistency:

| Asset | Protection |
|---|---|
| Main website | Imperva WAF |
| Agent portal | AWS CloudFront WAF |
| VPN portals | Okta SSO + AWS Global Accelerator |
| Travel portal | Direct AWS EC2 — no WAF |

The travel portal stood out immediately — a direct IP with no CNAME chain and no WAF intermediary, unlike every other subdomain.

### TLS Issues
Modern OpenSSL blocked connections due to legacy TLS renegotiation on the target. Worked around this by:
1. Using `openssl s_client -legacy_renegotiation`
2. Writing a custom Python `LegacyTLSAdapter` using `urllib3`

### Key Learning
Legacy TLS (`TLSv1.2` only, no secure renegotiation) is a common finding on older infrastructure. Modern clients block it by default requiring manual workarounds during testing.

---

## Phase 3 — Service Identification

### Tools Used
- `openssl s_client`
- Python3 HTTP requests
- Burp Suite Community

### Process
Confirmed the travel portal was running a Palo Alto GlobalProtect VPN portal by:
- Observing the login page in browser
- Capturing HTTP 302 redirect to `/global-protect/login.esp`
- Extracting TLS certificate confirming asset ownership (DigiCert, valid 2025–2026)
- Analysing HTTP response headers

### Headers of Note
```
X-Frame-Options: DENY
Strict-Transport-Security: max-age=31536000
Content-Security-Policy: script-src 'self' 'unsafe-inline'
```

The `unsafe-inline` CSP directive was noted as a weakness however the VDP program explicitly excluded CSP best practice findings.

---

## Phase 4 — PAN-OS Version Fingerprinting

### Tools Used
- `panos-scanner.py` (Bishop Fox / community fork)
- Manual ETag analysis
- Python3

### Process
Attempted to fingerprint the exact PAN-OS version using multiple methods:

1. **Static asset timestamps** — server returned no `Last-Modified` headers
2. **ETag analysis** — decoded ETag `691ce83a` as Unix timestamp → build date **November 18, 2025**
3. **Content length matching** — no public database covers builds beyond April 2024
4. **Server headers** — stripped by the server

The scanner's `version-table.txt` only covered builds up to April 2024 — 19 months behind the target's build date.

### Key Learning
PAN-OS version fingerprinting relies on community-maintained databases that lag behind actual releases. A recent build date (Nov 2025) suggests the target may be relatively up to date, making CVE correlation difficult without the exact version.

---

## Phase 5 — Broader Asset Discovery

### Tools Used
- `subfinder` on parent company domain
- `dig` for resolution
- Certificate transparency logs (`crt.sh`)

### Process
Expanded recon to the parent company domain and discovered 90+ subdomains. Investigated high-value targets including:

- **Code analysis tool** — resolved but firewalled (not publicly accessible)
- **Payment portal** — behind Imperva WAF
- **Dev API endpoint** — resolved to Azure but all ports refused
- **Auth flow portals** — protected behind PingOne + CloudFront
- **Streaming service** — resolved but firewalled

### Pattern Observed
Internal tools consistently resolved in DNS but were firewalled from the internet — good network segmentation practice. Public-facing assets were almost universally behind WAF solutions.

---

## Phase 6 — Burp Suite Testing

### Tools Used
- Burp Suite Community Edition
- Manual Repeater testing

### Process
Used Burp to manually test the GlobalProtect login endpoint for input reflection. Sent POST requests with test strings to parameters including `user`, `passwd`, `prot`, `server`, `inputStr`, and `action`.

The server consistently returned a 79-byte redirect response without processing input — no reflection found. The server-side validation rejected requests before reaching any processing logic.

### Key Learning
Burp Community lacks an active scanner — all testing must be manual. Always test input reflection before attempting XSS payloads. A server redirecting without processing input is not necessarily secure — it may require specific session state to reach vulnerable code paths.

---

## Phase 7 — Scope Review & Report Assessment

### Process
Reviewed the VDP policy carefully and found several findings were explicitly out of scope:

- TLS/SSL configuration issues
- CSP best practices
- Authentication flows
- Software version disclosure
- Rate limiting

This significantly narrowed the reportable attack surface.

### Finding
**Missing WAF on travel portal** — the only public-facing asset without WAF protection, running a Palo Alto GlobalProtect VPN portal directly on AWS EC2.

### Outcome
Could not confirm a specific vulnerable PAN-OS version — the version fingerprinting database was too outdated. Without a confirmed CVE match, the finding was classified as a security observation rather than a confirmed exploitable vulnerability. Decided not to submit to protect HackerOne Signal score as a new user.

---

## Tools & Techniques Summary

| Tool | Purpose |
|---|---|
| `subfinder` / `amass` | Passive subdomain enumeration |
| `dig +tcp` | DNS resolution through Tor |
| `openssl s_client` | TLS fingerprinting and HTTP testing |
| `proxychains` | Routing traffic through Tor (AnonSurf) |
| `nmap -sT` | TCP port scanning through Tor |
| Python3 LegacyTLSAdapter | Bypassing modern TLS restrictions |
| `panos-scanner.py` | PAN-OS version fingerprinting |
| Burp Suite Community | Manual HTTP request testing |
| `crt.sh` | Certificate transparency recon |

---

## Key Lessons Learned

1. **AnonSurf limitations** — Tor doesn't support UDP, breaking standard DNS queries and Nmap UDP scans. Use `+tcp` flag and proxychains for TCP-only tools.

2. **Version databases lag** — Public PAN-OS fingerprinting databases were 19 months behind the target. Passive version fingerprinting has real limitations against up-to-date targets.

3. **Missing WAF ≠ vulnerability** — Infrastructure observations without demonstrated exploitation don't meet most programs' bar for acceptance.

4. **Signal matters** — HackerOne's Signal system penalises low quality submissions. Don't submit findings you can't fully prove, especially as a new user.

5. **Good segmentation exists** — Most internal tools were firewalled from the internet despite resolving in DNS. Network segmentation is an effective defence.

---

## What I Would Do Differently

- Focus on finding input reflection points earlier in the process
- Set up a dedicated testing environment without AnonSurf for initial recon

---

*All target-specific details, screenshots, and identifying information have been redacted for public portfolio use.*
