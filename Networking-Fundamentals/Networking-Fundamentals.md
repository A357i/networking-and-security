# Networking Fundamentals

> Consolidated reference — one living file for the topic, not split per TryHackMe room.
> Update this in place as new rooms/labs touch networking concepts, rather than creating a new file each time.

---

## 0. What is Networking?
The practice of connecting devices so they can exchange data. Every device needs an address (IP), a way to reach others (routing), and an agreed "language" (protocols) — everything below is a piece of that puzzle.

## 0.1 Intro to LAN
- **LAN (Local Area Network)**: devices connected within a limited area (home, office) — typically one broadcast domain.
- **WAN (Wide Area Network)**: connects LANs across larger distances (the internet is the largest WAN).
- **Topologies**: star (all devices to central switch — most common today), bus, ring, mesh.

## 1. The OSI Model (7 Layers)

| # | Layer | Function | Example Protocols/Devices |
|---|-------|----------|----------------------------|
| 7 | Application | User-facing services | HTTP, DNS, FTP, SMTP |
| 6 | Presentation | Encoding, encryption, compression | TLS/SSL, JPEG |
| 5 | Session | Manages connections/sessions | NetBIOS, RPC |
| 4 | Transport | End-to-end delivery, reliability | TCP, UDP |
| 3 | Network | Logical addressing, routing | IP, ICMP, routers |
| 2 | Data Link | Physical addressing, framing | MAC, switches, ARP |
| 1 | Physical | Raw bit transmission | Cables, NICs, radio |

**Why it matters offensively:** most attacks map cleanly to a layer — ARP spoofing (L2), IP spoofing/routing attacks (L3), TCP session hijacking (L4), and most web/app attacks (L7). Knowing the layer tells you which tool category applies.

### TCP/IP Model (practical 4-layer version)
`Application → Transport → Internet → Link` — this is the model real traffic and tools (Wireshark, tcpdump) actually reflect; OSI is the conceptual teaching model, TCP/IP is what's implemented.

---

## 2. IP Addressing & Subnetting

- **IPv4**: 32-bit, four octets (e.g., `192.168.1.10`). **IPv6**: 128-bit, hex groups.
- **Private ranges (RFC1918)**: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — never routable on the public internet, common in internal pentest engagements.
- **CIDR notation**: `/24` = 255.255.255.0 = 256 addresses (254 usable). Subnet math is a core recon skill — knowing a target's CIDR tells you the scope of a network sweep.
- **NAT**: lets private IPs share a public IP for outbound traffic — relevant when you see one external IP fronting many internal hosts.

---

## 2.1 Packets & Frames
- **Frame**: Layer 2 (Data Link) unit — has source/destination MAC addresses.
- **Packet**: Layer 3 (Network) unit — has source/destination IP addresses, encapsulated inside a frame.
- **Encapsulation**: as data moves down the stack, each layer wraps the data with its own header (like nested envelopes). Decapsulation reverses this on receipt. Understanding this is essential for reading Wireshark captures — each layer's header is a separate section of the packet detail view.

## 2.2 Extending Your Network
- **Switch**: connects devices within a LAN, forwards frames based on MAC address (Layer 2).
- **Router**: connects different networks, forwards packets based on IP address (Layer 3) — this is what enables LAN-to-WAN (internet) connectivity.
- **Repeater/Access Point**: extends physical/wireless range without doing any addressing logic (Layer 1/2).

## 3. DNS (Domain Name System)

- Resolves domain names → IP addresses via a hierarchical query (root → TLD → authoritative server).
- **`/etc/hosts`** (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows) is checked **before** any DNS query — lets you locally override resolution, essential for testing vhosts/subdomains not yet in public DNS.
- **Record types worth knowing**: `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias), `MX` (mail), `TXT` (arbitrary text — often SPF/verification), `NS` (nameservers).
- **Subdomain takeover**: occurs when a `CNAME` points to a cloud resource (S3 bucket, Azure/Heroku endpoint) that's been deleted/unclaimed. An attacker re-registers that resource name and now serves content on the victim's subdomain. Found via dangling-CNAME scanning.

---

## 4. Virtual Hosting & vhost Fuzzing

- One server IP can host multiple domains, distinguished by the HTTP `Host` header — this is virtual hosting.
- **`ffuf`** (or `gobuster vhost`) brute-forces the `Host` header against a wordlist to reveal vhosts the server responds to but doesn't publicly advertise — a core active-recon technique once you have an IP but suspect hidden sites.

---

## 5. Common Ports & Protocols

| Port | Protocol | Notes |
|------|----------|-------|
| 21 | FTP | Often anonymous login misconfig |
| 22 | SSH | Key vs password auth, common brute-force target |
| 23 | Telnet | Unencrypted — legacy/IoT risk |
| 25 | SMTP | Mail relay, open-relay misconfig |
| 53 | DNS | TCP for zone transfers, UDP for queries |
| 80/443 | HTTP/HTTPS | Primary web attack surface |
| 445 | SMB | Windows file sharing, common lateral-movement vector |
| 3389 | RDP | Remote desktop, brute-force/exposed-RDP target |

---

## 6. Recon Classification

- **Passive recon**: no direct interaction with the target — WHOIS, cached content, public DNS records, OSINT/search engines. Leaves no footprint on target logs.
- **Active recon**: direct interaction — port scans (`nmap`), banner grabbing, vhost fuzzing. Noisier, more detectable, but far richer data. This is the natural next phase after passive recon.

---

## 7. Where This Connects to the Kill Chain

Networking fundamentals underpin the first stage of the **Cyber Kill Chain** (Reconnaissance) most directly — subnetting and DNS knowledge define your scope, port/protocol knowledge tells you what's worth attacking, and vhost/subdomain techniques extend your visible attack surface before you ever touch Weaponization or Delivery.

---

## Open Questions / To Expand
- [ ] Add IPv6 addressing detail once covered in a room
- [ ] Add routing protocol basics (static vs dynamic) when relevant
- [ ] Add firewall/IDS-IPS section
