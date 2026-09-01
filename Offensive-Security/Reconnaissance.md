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

### 3.4 Traceroute
- Traces the route packets take from your system to a target host — discovers the IP addresses of routers (hops) along the path, and how many sit between you and the destination. Useful for understanding network topology, spotting where filtering/latency occurs, and mapping infrastructure.
- **The route isn't fixed**: routers use dynamic routing protocols (BGP, OSPF) that adapt to network changes, and modern networks use load balancing and anycast routing — so packets can take different paths even across consecutive runs of the same command.
- **Syntax**: Linux/macOS `traceroute <target>`; Windows `tracert <target>`; IPv6 `traceroute -6 MACHINE_IPV6` or standalone `traceroute6`.
- **How it works**: exploits the **TTL field** in the IP header. Each router decrements TTL by 1 before forwarding; when TTL hits 0, that router drops the packet and sends back an **ICMP Time-to-Live Exceeded** message. By sending packets with incrementally increasing TTL starting at 1, `traceroute` forces each successive router to reveal itself — TTL=1 gets dropped (and answered) by the first router, TTL=2 by the second, and so on until the packet reaches the destination.
- Some routers are configured **not** to send Time-to-Live Exceeded messages (common in secure environments specifically to hinder recon) — these show up as `*` in the output.
- On Linux, `traceroute` sends **UDP** datagrams by default. Switch to **TCP-based** tracing (useful for bypassing UDP filters) with `traceroute -T <target>`; switch to **ICMP-based** tracing with `traceroute -I <target>`.
- **Reading output**: each numbered line = one hop. Three packets are sent per TTL value, so a line can show up to 3 IPs and 3 RTTs — different IPs on the same hop number usually just means load balancing sent each of the three packets down a slightly different path. The final line matching the destination IP means the trace completed; every line before it is an intermediate router.
- **Route variability is normal, not a bug**: two traceroutes to the same target run minutes apart can differ meaningfully in hop count (one real example: 14 hops vs. 26 hops to the same destination) — this is expected for traffic crossing external networks, especially through CDNs like Cloudflare/Akamai that use anycast and load balancing to optimize paths.
- **Three things worth internalizing**: (1) hop count depends on *when* you run the command — no guarantee of a stable path even minutes apart; (2) some routers return public IPs worth examining depending on engagement scope, though they may belong to third parties, not the target; (3) routers that don't reply at all could mean rate limiting, firewall rules, or deliberate ICMP suppression — not necessarily "nothing there."
- **Additional techniques**: `mtr <target>` ("My Traceroute") — real-time, continuous view combining traceroute with ping-like stats (packet loss + latency per hop).

### 3.5 Telnet
- **TELNET** (Teletype Network), from 1969 — communicates with a remote system via a command-line interface, default port 23. Sends **all data in cleartext**, including usernames/passwords, making it trivial to intercept credentials. The secure replacement is **SSH**, which encrypts everything and is the modern standard for remote CLI access.
- **Recon value despite being obsolete for admin use**: because it operates over TCP, `telnet` can connect to *any* TCP port and show you the server's response — this is **banner grabbing**. Connect to a service, read the initial response (the "banner") it sends back; banners frequently reveal the exact software name and version running on that port, which can then be cross-referenced against CVE/Exploit-DB.
- Install if missing (Debian/Ubuntu): `apt install telnet`. In practice, `netcat`/`nc` and `curl` are generally preferred — same core idea, more flexibility.
- **Example (HTTP banner grab)**: `telnet <target> 80`, then type `GET / HTTP/1.1`, then `host: example`, then press Enter twice. Response includes a `Server:` header revealing the web server + version.
- **Same principle applies to any TCP-based service**: connect to an SMTP/POP3 server and use the relevant protocol commands; an FTP server on port 21 typically sends its banner immediately with no commands needed at all. Connect → read what's sent back → optionally issue protocol-specific commands.
- **Limitation**: telnet can't handle encrypted connections, and modern services increasingly enforce TLS (SMTPS on 465, HTTPS on 443). For HTTPS, use `curl --head https://<target>` or `openssl s_client -connect <target>:443`; for other TLS-wrapped services, `openssl s_client` or `ncat --ssl`.

### 3.6 Netcat
- **`nc`** — a versatile TCP/UDP networking utility. Works as a **client** (connects to a listening port) or a **server** (listens on a port you choose) — that dual capability makes it useful for banner grabbing, port probing, simple file transfers, and basic client-server communication. Modern builds (`ncat`, from the Nmap project) add IPv6 and SSL support, making them more capable than legacy `telnet`.
- **Banner grabbing** works identically to telnet: `nc <target> <port>`, then issue the protocol-appropriate request (e.g. `GET / HTTP/1.1` + `host: example`, sometimes needing Shift+Enter after the GET line). Same logic applies across FTP/SMTP/etc. — some services banner immediately, others need a command first.
- **Listening (server mode)**: `nc -vnlp 1234` starts listening on port 1234; from the other side, `nc <target> 1234` connects. Once connected, anything typed on either side is transmitted to the other — useful for testing connectivity or basic data transfer during an engagement.

| Flag | Meaning |
|---|---|
| `-l` | Listen mode |
| `-p` | Specify the port (must sit directly before the port number) |
| `-n` | Numeric only — skips DNS resolution and its warnings |
| `-v` / `-vv` | Verbose / very verbose output |
| `-k` | Keep listening after the client disconnects |
| `-6` | Listen on IPv6 |

- Ports below 1024 require root privileges to listen on. For encrypting sensitive transfers, use `ncat --ssl` or pair `nc` with `stunnel`.

### 3.7 Putting It All Together
Five core active-recon tools, used together rather than in isolation: **browser** (Developer Tools reveal server tech, headers, JS sources, cert details) → **`ping`** confirms a target is reachable and gives TTL-based OS clues → **`traceroute`** maps the network path and reveals intermediate routers/filtering points → **`telnet`**/**`netcat`** connect to individual ports to grab banners and identify running services + versions. A typical flow: `ping` to confirm the host is alive, `traceroute` to understand the path, then `nc`/`curl -I` to probe specific ports — preferring `curl`/`nc` over `telnet` for banner grabbing since they handle TLS and offer more flexibility.

**Where this leads next**: `nmap` automates and extends host discovery + port scanning far beyond what `ping`/`nc` can do manually — that's the natural next room. Deeper web-recon technique lives in "Walking An Application"; stealth/evasion topics (slow timing, proxy chaining, blended traffic) live in the stealth-scanning and Burp Suite rooms.

---

## Open Questions / To Expand
- [x] Active Recon room complete (browser recon, ping, traceroute, telnet, netcat)
- [ ] Add Nmap/service enumeration once that room is done
- [ ] Add "Walking An Application" deep web-recon notes once covered
- [ ] Add stealth scanning / evasion techniques (slow timing, proxy chaining) once covered
