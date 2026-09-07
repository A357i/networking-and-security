# Nmap Live Host Discovery

## Overview
Nmap (Network Mapper) is free, open-source software released under the GPL license, created by Gordon Lyon (Fyodor). It is the industry-standard tool for mapping networks, identifying live hosts, and discovering running services. Its scripting engine extends functionality further — from fingerprinting services to exploiting vulnerabilities.

A full Nmap scan typically goes through these steps (many optional, depending on command-line arguments):
1. Enumerate targets
2. Discover live hosts
3. Reverse-DNS lookup
4. Scan ports
5. Detect versions
6. Detect OS
7. Traceroute
8. Scripts
9. Write output

This room focuses on step 2 — **discovering live hosts** — before any port scanning happens. Understanding host discovery matters because scanning an offline host wastes time and generates unnecessary noise.

## Network Terminology
- A **network segment** is a group of computers connected using a shared medium (e.g. an Ethernet switch or Wi-Fi access point).
- A **subnetwork (subnet)** is usually equivalent to one or more network segments connected to the same router. The segment refers to the physical connection; the subnet refers to the logical connection.
- A subnet has its own IP address range and connects to a larger network via a router. A **firewall** may sit between segments, enforcing security policy on traffic passing between them.
- Subnet mask sizes:
  - `/16` → `255.255.0.0` — accommodates around 65,000 hosts.
  - `/24` → `255.255.255.0` — accommodates around 250 hosts.

## TCP/IP Layers Used for Host Discovery
| Layer | Protocols usable for discovery |
|---|---|
| Link Layer | ARP |
| Network Layer | ICMP |
| Transport Layer | TCP, UDP |

Mapping to OSI/TCP-IP models:

| OSI Layer | TCP/IP Layer | Protocols |
|---|---|---|
| 7 Application | Application | HTTP, HTTPS, SMTP, POP3, IMAP, SSH, FTP, SNMP, Telnet, RDP, ... |
| 6 Presentation | Application | (TLS/SSL sit here conceptually) |
| 5 Session | Application | |
| 4 Transport | Transport | TCP, UDP |
| 3 Network | Network | IPv4, IPv6, ICMP, IPsec |
| 2 Data Link | Link | ARP, Ethernet (802.3), WiFi (802.11), DSL, Bluetooth, ... |
| 1 Physical | Link | |

- **ARP** has one purpose: send a frame to the broadcast address on the network segment, asking the computer with a specific IP to respond with its MAC (hardware) address.
- **ICMP** has many types. ICMP ping uses Type 8 (Echo) and Type 0 (Echo Reply). If pinging a system on the same subnet, an ARP query should precede the ICMP Echo.
- **TCP/UDP**: although transport-layer protocols, a scanner can send a specially crafted packet to common TCP/UDP ports to check whether the target responds. Efficient, especially when ICMP Echo is blocked.

## Target Specification
| Type | Example | Result |
|---|---|---|
| List | `MACHINE_IP scanme.nmap.org example.com` | Scans 3 IP addresses |
| Range | `10.11.12.15-20` | Scans 6 addresses: `10.11.12.15` ... `10.11.12.20` |
| Subnet | `MACHINE_IP/30` | Scans 4 IP addresses |

```bash
# Provide a file as input for target list
nmap -iL list_of_hosts.txt

# List hosts Nmap would scan without actually scanning them (attempts reverse-DNS)
nmap -sL TARGETS

# Suppress reverse-DNS resolution
nmap -sL TARGETS -n
```

## ARP Host Discovery
- ARP scan is possible **only if you are on the same subnet** as the targets.
- On Ethernet (802.3) / Wi-Fi (802.11), you need the MAC address of a system before communicating with it — obtained via ARP query.
- A host that replies to an ARP query is up.
- ARP is a link-layer protocol and **ARP packets are bound to their subnet** — they are not routed across subnets. If you're on a different subnet than the target, your scanner's packets go through the default gateway/router instead, and ARP queries won't cross that router.

```bash
# ARP scan only, no port scan
nmap -PR -sn TARGETS
```
`-PR` indicates an ARP scan; `-sn` means host discovery only, no port scan.

Example: `nmap -PR -sn 10.200.6.0/24` — sends ARP requests to all target computers; online ones reply with an ARP reply. In Wireshark, this shows as an ARP request (source MAC known, destination broadcast) followed by an ARP reply from the live host, with requests sent sequentially starting from the first host in the subnet.

ARP scanning is particularly useful during **post-exploitation** and internal network enumeration — once inside a network, ARP scans quickly and reliably identify other live hosts on the same local segment. Because ARP operates at Layer 2 and is often not filtered by firewalls, it's effective for red-team operations and internal penetration testing.

## ICMP Host Discovery
Default behavior when no host discovery options are provided:
1. **Privileged user, local network (Ethernet)** → Nmap uses ARP requests.
2. **Privileged user, outside local network** → Nmap uses ICMP echo requests, TCP ACK to port 80, TCP SYN to port 443, and ICMP timestamp requests.
3. **Unprivileged user, outside local network** → Nmap resorts to a TCP 3-way handshake, sending SYN packets to ports 80 and 443.

Nmap by default uses a ping scan to find live hosts, then scans only live hosts. To discover hosts without port-scanning:
```bash
nmap -sn TARGETS
```

### ICMP Echo (Type 8/0)
```bash
nmap -PE -sn TARGET
```
Sends an ICMP echo request, expects an ICMP echo reply if the target is online. Not always reliable — many firewalls block ICMP echo; newer Windows versions block it by default via host firewall. Remember: an ARP query precedes an ICMP request if the target is on the same subnet.

```bash
sudo nmap -PE -sn 10.200.6.0/24
```

### ICMP Timestamp (Type 13/14)
```bash
nmap -PP -sn TARGET
```
Because ICMP echo tends to be blocked, ICMP Timestamp or Address Mask requests are alternatives. Nmap sends a timestamp request (Type 13) and checks for a Timestamp reply (Type 14).

### ICMP Address Mask (Type 17/18)
```bash
nmap -PM -sn TARGET
```
Nmap sends an address mask query (Type 17) and checks for an address mask reply (Type 18).

**Key lesson**: different ICMP types can be blocked independently. In one lab run, ICMP Echo (`-PE`) and default scan found 3 hosts up, but ICMP Address Mask (`-PM`) found 0 hosts up (0 hosts up scanned in 52.14 seconds) even though hosts were live — the target/firewall was blocking that specific ICMP type. **It is essential to learn multiple approaches; if one packet type is blocked, choose another to discover the target network.**

## TCP/UDP Host Discovery

### TCP SYN Ping
```bash
nmap -PS[port(s)] -sn TARGET
```
Sends a packet with the SYN flag set to TCP port 80 by default. An open port replies with SYN/ACK; a closed port replies with RST. Either response confirms the host is up — the specific port state doesn't matter here.

- `-PS21` → target port 21
- `-PS21-25` → target ports 21 through 25
- `-PS80,443,8080` → target ports 80, 443, and 8080

**Privileged users** (root/sudoers) can send TCP SYN packets and don't need to complete the 3-way handshake even if the port is open. **Unprivileged users** must complete the full 3-way handshake if the port is open.

```bash
nmap -PS -sn 10.200.6.0/24
```

### TCP ACK Ping
```bash
nmap -PA[port(s)] -sn TARGET
```
Sends a packet with the ACK flag set. **Requires privileged user** — an unprivileged user triggers a 3-way handshake attempt instead. By default, port 80 is used. Syntax for ports is the same as `-PS` (`-PA21`, `-PA21-25`, `-PA80,443,8080`).

Any TCP packet with the ACK flag set should receive a TCP packet with the **RST** flag set in response — the target responds with RST because the ACK packet isn't part of any ongoing connection. This response is used purely to detect that the host is up (works whether the port is open or closed).

```bash
sudo nmap -PA -sn 10.200.6.0/24
```

### UDP Ping
```bash
nmap -PU[port(s)] -sn TARGET
```
Unlike TCP SYN ping, sending a UDP packet to an **open** port is not expected to elicit a reply. However, sending a UDP packet to a **closed** UDP port triggers an **ICMP port-unreachable** (Type 3, Code 3) response — this indirectly confirms the target is online. Port syntax matches `-PS`/`-PA`.

```bash
sudo nmap -PU -sn 10.200.6.0/24
```

### Masscan (mentioned alongside UDP ping)
Masscan uses a similar discovery approach but is far more aggressive in packet generation rate, completing network scans quickly.
```bash
masscan 10.200.6.0/24 -p443
masscan 10.200.6.0/24 -p80,443
masscan 10.200.6.0/24 -p22-25
```
Not installed on the AttackBox by default — install with `apt install masscan`.

## Reverse-DNS Lookup
Reverse DNS (rDNS) resolves an IP address to a hostname (the opposite of normal DNS) — e.g. "What domain name is assigned to 10.200.6.15?" instead of "What is the IP of example.com?".

```bash
# Force reverse-DNS lookup for all discovered hosts (even offline ones)
nmap -R TARGET

# Disable reverse-DNS lookups
nmap -n TARGET

# Use a specific DNS server
nmap -R --dns-servers DNS_SERVER TARGET
```
By default, Nmap looks up online hosts only. `-R` forces lookups even for offline hosts. rDNS helps identify system roles (e.g. `mail.company.local`, `dc01.domain.com`) and understand network structure during recon — but records aren't always configured or accurate, so results may be missing/misleading, and it can slightly slow scans due to additional DNS queries.

## Full Command Reference

| Scan Type | Example Command |
|---|---|
| ARP Scan | `sudo nmap -PR -sn 10.200.6.0/24` |
| ICMP Echo Scan | `sudo nmap -PE -sn 10.200.6.0/24` |
| ICMP Timestamp Scan | `sudo nmap -PP -sn 10.200.6.0/24` |
| ICMP Address Mask Scan | `sudo nmap -PM -sn 10.200.6.0/24` |
| TCP SYN Ping Scan | `sudo nmap -PS22,80,443 -sn 10.200.6.0/30` |
| TCP ACK Ping Scan | `sudo nmap -PA22,80,443 -sn 10.200.6.0/30` |
| UDP Ping Scan | `sudo nmap -PU53,161,162 -sn 10.200.6.0/30` |

| Option | Purpose |
|---|---|
| `-n` | No DNS lookup |
| `-R` | Reverse-DNS lookup for all hosts |
| `-sn` | Host discovery only (no port scan) |

**Remember**: add `-sn` if only interested in host discovery without port-scanning. Omitting `-sn` lets Nmap default to scanning live hosts for open ports afterward.

## Key Takeaways
- Effective live host discovery lays the foundation for accurate scoping and efficient follow-up scans in any penetration test.
- No single discovery method is fully reliable — firewalls and host configurations can block ARP (off-subnet only anyway), specific ICMP types, or TCP/UDP probes independently. Combining multiple techniques ensures accurate results.
- ARP is confined to the local subnet; ICMP/TCP/UDP techniques are needed for cross-subnet discovery.
- Next step in the Nmap room series: **Nmap Basic Port Scans**, which builds on this host-discovery foundation.
