# Methodology

> Covers: Dive Into Pentesting, Guided Pentest Infrastructure, Search Skills, Cyber Kill Chain, Penetration Testing Frameworks

---

## 1. Dive Into Pentesting
A pentest engagement generally moves through: **Scoping** (what's in/out of bounds, written authorization) → **Reconnaissance** → **Enumeration** → **Exploitation** → **Post-Exploitation** (privilege escalation, lateral movement) → **Reporting**. The report is the actual client deliverable — findings without clear reproduction steps and business impact aren't useful to a client.

## 2. Guided Pentest Infrastructure
Setting up a working attack environment: an attack box (Kali/Parrot or similar), isolated network segmentation from your host machine, and a way to capture/organize output as you go (this is exactly why the note structure in this repo matters — infra without documentation discipline falls apart on real engagements).

## 3. Search Skills
Effective OSINT/search techniques for recon: Google dorking (`site:`, `filetype:`, `intitle:`), searching for leaked credentials/config files, using specialized search engines (Shodan, Censys) for exposed services. The skill here is knowing what to search for, not just how to search.

## 4. The Cyber Kill Chain
Lockheed Martin's 7-stage model of an attack:
1. **Reconnaissance** — gathering info on the target
2. **Weaponization** — building the exploit/payload
3. **Delivery** — getting the payload to the target (phishing, drive-by, etc.)
4. **Exploitation** — triggering the vulnerability
5. **Installation** — establishing persistence
6. **Command & Control (C2)** — remote control channel established
7. **Actions on Objectives** — the attacker's actual goal (data theft, ransomware, etc.)

**Defensive value**: breaking the chain at *any* stage stops the attack — this is why defenders don't need to catch everything, just one link.

## 5. Penetration Testing Frameworks
| Framework | Focus |
|---|---|
| **PTES** (Penetration Testing Execution Standard) | End-to-end methodology, pre-engagement through reporting |
| **OSSTMM** | Measurable security testing metrics |
| **OWASP Testing Guide** | Web application-specific testing methodology |
| **NIST SP 800-115** | Technical guide to information security testing |

Frameworks matter because they standardize scope/methodology so nothing gets missed and reports are consistent and defensible.

---

## Open Questions / To Expand
- [ ] Add specific OSINT tool syntax to Tools-Cheatsheets once used hands-on
