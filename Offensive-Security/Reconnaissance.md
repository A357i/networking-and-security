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

## 3. Active Reconnaissance (TryHackMe: Active Recon room, in progress)

### 3.1 What Changes vs. Passive
Active recon means directly interacting with the target — probing, connecting, sending traffic — rather than only reading public data. That interaction leaves traces: log entries, IDS alerts, WAF blocks, honeypot triggers. Prerequisite knowledge: TCP/UDP/ICMP and port basics (see `Networking-Fundamentals.md`).

**Modern context worth internalizing**: most orgs now run CDNs (Cloudflare, Akamai) + WAFs + zero-trust logging, so active recon is riskier/more detectable than it used to be. IPv6 hosts often respond to `ping6` but filter ICMPv4. HTTPS dominance makes plaintext protocols mostly obsolete for real traffic — but tools like `telnet` are still worth understanding because they teach banner-grabbing and cleartext protocol interaction fundamentals that `netcat`/`curl` build on.

**Critical rule**: never perform active recon without explicit, signed authorization (pentest contract or bug bounty scope) — unauthorized probing is illegal in most jurisdictions.

**Red team vs. blue team framing**: offense tries to blend probe traffic in with normal browsing (realistic User-Agent, slower timing); defense watches access logs, firewall logs, WAF events, and IDS alerts for exactly those probes. Worth holding both lenses at once.

### 3.2 Web Browser as a Recon Tool
The browser is one of the least suspicious active-recon tools — it's present everywhere and its traffic blends with normal use.

- **Ports**: browsers default to TCP 80 (HTTP, now rare — most sites redirect to HTTPS) and TCP 443 (HTTPS, the modern standard). Non-standard ports can be hit directly in the URL (`https://target.com:8443/`).
- **HTTP/3 & QUIC**: QUIC combines TCP+TLS's functions into one protocol running over **UDP port 443** — faster, more reliable than classic TCP+TLS. Visible in DevTools' Network tab as protocol `h3`.
- **Developer Tools** (`Ctrl+Shift+I` / `Option+Cmd+I`):
  - **Network tab**: every request/response in real time — headers like `Server`, `X-Powered-By`, `Content-Security-Policy`, plus timing, status codes, cookies.
  - **Console tab**: run JS snippets directly, view errors, interact with the DOM.
  - **Sources tab**: browse loaded JS/CSS/HTML — JS files frequently contain hardcoded API endpoints, directory structures, internal service references, and dev comments never meant to be public. One of the most practically useful recon techniques available through the browser alone.
  - **Application tab (Storage)**: cookies, LocalStorage, SessionStorage — sometimes contain session tokens, API keys, or tracking IDs exposed client-side.
  - **Security tab**: certificate issuer, validity period, and **SANs (Subject Alternative Names)** — SANs frequently reveal additional subdomains belonging to the same org.
- **Useful extensions**:
  - **FoxyProxy** — switch between proxies (Burp Suite, ZAP, SOCKS5 tunnels) for intercepting/routing traffic.
  - **User-Agent Switcher and Manager** — emulate other browsers/OS/devices to surface mobile-specific endpoints or version-specific behavior (modern WAFs/CDNs do watch for rapid UA changes though).
  - **Wappalyzer** (and alternatives BuiltWith, WhatRuns, Library Detector) — passively fingerprints the tech stack: CMS, web server + version, JS frameworks, analytics, CDN, database hints.
- Even "normal" browsing can trip WAF/EDR behavioral rules: rapid page loads, modified headers, frequent DevTools use, abnormal User-Agent strings. The goal is to mimic legitimate user behavior as closely as possible.

### 3.3 Ping (ICMP)
- Named for the sonar analogy: send a signal, listen for the echo. Uses **ICMP**: an **Echo Request** (type 8) out, an **Echo Reply** (type 0) back if the target is reachable and permits it. Lightweight and fast — the standard first check before deeper scanning.
- **Syntax**: Linux/macOS use `-c <n>` for packet count (`ping -c 5 10.113.178.91`); Windows uses `-n <n>`. Omitting the count on Linux pings indefinitely (`Ctrl+C` to stop). Force an IP version with `-4`/`-6` (or standalone `ping6` on some systems). Can ping a hostname directly — DNS resolution happens first.
- **Interpreting success**: 0% packet loss = clean path; a low RTT (~0.5ms) suggests the target is on the same local network.
- **TTL (Time To Live)** — despite the name, this is actually a **hop count**, not a timer: each router along the path decrements it by 1, and the packet is dropped at 0. The OS sets the *initial* TTL, which makes it useful for OS fingerprinting — **Linux typically starts at 64, Windows at 128**. Caveat: intermediate routers decrement the value before it reaches you, so a received TTL of 58 likely means a Linux host **6 hops away**, not evidence of a different OS. Always account for hop count before concluding anything from TTL.
- **Interpreting no reply / "Destination Host Unreachable"**: several possible causes — target is off/crashed/still booting, a router/firewall is blocking ICMP Echo Requests, target is behind NAT that drops ICMP, Windows Firewall blocks ping by default on most versions, or a corporate firewall/cloud provider (AWS/Azure/GCP)/modern WAF-CDN is blocking ICMP outright.

| Result | Most likely meaning | Next step |
|---|---|---|
| Fast replies, low/no packet loss | Target online, allows ICMP | Proceed to port scanning |
| "Destination Host Unreachable" | Target down or no route exists | Check if machine is powered on |
| 100% packet loss, no error message | ICMP filtered/blocked | Try TCP/UDP host discovery with Nmap |
| High latency or heavy loss | Congestion, distance, or filtering | Investigate the path with `traceroute` |

- Quick syntax notes: `-s` sets the size of the ICMP echo request's data payload; the ICMP header itself is **8 bytes**.

---

## Open Questions / To Expand
- [ ] Active Recon room in progress — still to add: `traceroute`/`mtr` path mapping, `telnet` banner grabbing, `netcat` banner grabbing/port probing (Tasks 4+)
- [ ] Add Nmap/service enumeration once that room is done
