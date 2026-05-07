# Web Application Recon — Bug Bounty Writeup

Bug bounty programs are initiatives where companies publicly invite ethical security
researchers to find and responsibly report weaknesses in their systems legally and
with permission. This repository documents a real engagement I conducted on a live
program through HackerOne.

---

## Goal

The goal of this engagement was to map out everything a company exposes to the
internet, identify any weaknesses in how those assets are protected, and produce a
finding that could be responsibly reported to the company.

---

## What I Did

The engagement was carried out in seven structured phases:

**1. Finding all the doors**
I started by discovering every website, portal, and service the company makes
publicly available — like finding every entrance to a building before deciding
which ones to inspect more closely.

**2. Checking who's guarding each door**
For every asset found, I investigated what security protections were in place.
Most assets were shielded behind enterprise-grade web application firewalls.
One was not.

**3. Identifying what's behind the unprotected door**
The unprotected asset turned out to be a VPN login portal a gateway into the
company's internal network sitting directly on the internet with no firewall
in front of it, unlike every other public-facing system.

**4. Determining the software version**
I attempted to identify exactly which version of the VPN software was running,
to cross-reference it against known security vulnerabilities. This required
working around several technical obstacles the server presented.

**5. Expanding the investigation**
I broadened the search to the parent company's entire online footprint,
discovering 90+ additional assets and assessing their exposure. Most internal
tools were inaccessible from the internet good security practice but the
pattern of the unprotected portal remained the standout finding.

**6. Testing for input vulnerabilities**
Using industry-standard testing tools, I manually probed the login portal for
weaknesses that could allow an attacker to manipulate the system testing
whether the application processed user-supplied input in unsafe ways.

**7. Scope review and reporting decision**
I carefully reviewed the program's rules to determine what could be formally
reported. Several findings were explicitly out of scope. The key remaining
finding the unprotected portal could not be tied to a specific confirmed
vulnerability without an exact software version match.

---

## Key Finding

One public-facing asset a VPN login portal had no firewall protection,
while every other system belonging to the organization sat behind enterprise
security solutions. Think of it as every entrance to a building having a
security guard except one, which is left completely open.

The software running on that portal could not be fingerprinted precisely enough
to confirm a specific known vulnerability, due to the version database being
nearly two years out of date.

---

## Outcome & Professional Judgement

Rather than submitting an unconfirmed finding, I made the deliberate decision
**not to report**. On HackerOne, new researchers are scored on the quality of
their submissions submitting an observation without proof of exploitability
would have damaged my standing on the platform. This reflects a core principle
of professional security work: only report what you can demonstrate, not what
you suspect.

---

## Skills Demonstrated

- **Structured investigation** — seven-phase methodology from initial mapping
  through to reporting decision, mirroring real penetration testing engagements
- **Technical problem-solving** — overcame multiple obstacles including network
  restrictions and software compatibility issues to complete the assessment
- **Breadth of tooling** — used tools standard across the professional security
  industry for passive reconnaissance, traffic analysis, and manual testing
- **Professional judgement** — understood when *not* to act is the right call
- **Clear documentation** — produced a full written account of methodology,
  findings, and reasoning suitable for professional review

---

## Why This Matters

This was a real engagement against a live target, conducted legally within the
parameters of an authorized program. It demonstrates the ability to independently
scope, execute, and document a security assessment the foundation of
professional penetration testing and security consulting.

---

*All target-specific details, company names, and identifying information have
been redacted for responsible public disclosure.*
