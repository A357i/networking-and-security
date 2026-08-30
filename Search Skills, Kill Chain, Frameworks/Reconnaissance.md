# Reconnaissance

> Covers: Passive Reconnaissance, the TakeOver challenge (applied case study)
> Protocol-level theory (DNS, vhosts) lives in `Networking-Fundamentals/` — this file is the applied/offensive angle.

---

## 1. Passive vs. Active Recon
- **Passive**: no direct interaction with the target infrastructure — WHOIS lookups, cached pages, public DNS records, OSINT via search engines and social media, certificate transparency logs. Leaves zero footprint on target-side logs.
- **Active**: direct interaction — port scanning (`nmap`), banner grabbing, vhost fuzzing. Detectable, but yields live/current data passive sources can't.
- **Rule of thumb**: exhaust passive options first — you'd be surprised how much scope-relevant info is public before you ever send a single active packet.

## 2. Applied Case Study: TakeOver Challenge
Walkthrough of the technique chain used:
1. **Target had a subdomain** with a `CNAME` DNS record pointing to a cloud resource (e.g. an S3 bucket or app-hosting endpoint) that had since been deleted.
2. Used **DNS resolution** understanding to confirm the CNAME still pointed at the now-unclaimed resource name.
3. Used **`/etc/hosts` overrides** to test how a target server responded when a `Host` header didn't match public DNS — confirming vhost behavior locally before touching production DNS.
4. Ran **`ffuf`** against the `Host` header to enumerate additional vhosts the server responded to but didn't publicly list.
5. **Subdomain takeover**: because the CNAME still pointed at the deleted cloud resource, re-registering that resource name under an attacker-controlled account would let the attacker serve arbitrary content on the victim's subdomain — a real, common misconfiguration in bug bounty programs.

**Lesson**: this challenge is a good example of chaining fundamentals (DNS theory) into an actual finding — no single "advanced" technique was needed, just correctly applied basics.

---

## Open Questions / To Expand
- [ ] Add Active Recon section (nmap, service enumeration) once that room is done — natural next entry here
