# Nmap Basic Port Scans

Second room in the Nmap series (after [[nmap-host-discovery]]). Covers TCP connect scan, TCP SYN scan, UDP scan, and options for specifying ports, scan rate, and parallel probes.

## TCP and UDP Ports
A TCP or UDP port identifies a network service running on a host, the same way an IP address identifies the host itself. A server provides a network service and adheres to a specific protocol (e.g. serving web pages, responding to DNS queries). A port is usually linked to a service by convention — e.g. an HTTP server binds to TCP port 80 by default, and TCP port 443 if it supports SSL/TLS (though an admin could choose other ports). **No more than one service can listen on any given TCP or UDP port on the same IP address.**

### Port States (simplified)
- **Open** — a service is listening.
- **Closed** — no service is listening.

### Port States (Nmap's full six, accounting for firewalls)
| State | Meaning |
|---|---|
| Open | A service is listening on the specified port. |
| Closed | No service is listening, but the port is reachable (not blocked by a firewall/security appliance). |
| Filtered | Nmap cannot determine open vs closed because the port is not accessible — usually a firewall blocking Nmap's packets or blocking responses. |
| Unfiltered | Nmap cannot determine open vs closed even though the port is accessible. Encountered when using an ACK scan (`-sA`). |
| Open\|Filtered | Nmap cannot determine whether the port is open or filtered. |
| Closed\|Filtered | Nmap cannot determine whether the port is closed or filtered. |

## TCP Header and Flags (RFC 793)
The TCP header is the first 24 bytes of a TCP segment: source port and destination port (16 bits/2 bytes each), sequence number (32 bits), acknowledgement number (32 bits), then data offset/reserved/flags/window, checksum/urgent pointer, options/padding — 6 rows of 32 bits = 24 bytes total.

Nmap can set or unset these TCP flags (setting a flag bit means setting its value to 1):
| Flag | Meaning |
|---|---|
| **URG** | Urgent flag — indicates the urgent pointer field is significant; a segment with URG set is processed immediately without waiting for previously sent segments. |
| **ACK** | Acknowledgement flag — indicates the acknowledgement number is significant; used to acknowledge receipt of a TCP segment. |
| **PSH** | Push flag — asks TCP to pass the data to the application promptly. |
| **RST** | Reset flag — used to reset the connection. A device such as a firewall might send it to tear down a TCP connection; also used when data is sent to a host with no service listening on the receiving end. |
| **SYN** | Synchronise flag — used to initiate a TCP 3-way handshake and synchronise sequence numbers with the other host. The sequence number should be set randomly during connection establishment. |
| **FIN** | The sender has no more data to send. |

## TCP Connect Scan (`-sT`)
```bash
nmap -sT TARGET
```
Works by completing the full TCP 3-way handshake:
1. Client sends SYN.
2. Server responds SYN/ACK if the port is open.
3. Client completes the handshake with ACK.

Since we only care whether the port is open (not in maintaining a connection), the connection is torn down immediately after its state is confirmed, by sending RST/ACK.

**Key point**: if you are **not a privileged user** (root or sudoer), TCP connect scan is the **only** option available to discover open TCP ports — SYN scan requires privileges.

In a Wireshark capture, TCP connect scan shows the full pattern per port: SYN → SYN/ACK → ACK → RST/ACK (open port), repeated for all 1000 default ports scanned.

```bash
nmap -sT 10.113.152.172
```
Example output:
```
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.25 seconds
```

Useful modifiers:
- `-F` — fast mode, scans 100 most common ports instead of the default 1000.
- `-r` — scan ports in consecutive order instead of random order (useful for testing whether ports open consistently, e.g. during a target boot sequence).

## TCP SYN Scan (`-sS`)
```bash
sudo nmap -sS TARGET
```
- Default scan mode when running Nmap as a **privileged user** (root/sudo) — reliable and the standard choice.
- Does **not** need to complete the TCP 3-way handshake — it tears down the connection immediately upon receiving a response from the server (sends RST after SYN/ACK, instead of completing with ACK).
- Because no TCP connection is ever fully established, the scan is **less likely to be logged** by the target.
- Discovers the same open ports as a connect scan, but **without ever fully connecting**.

In Wireshark: TCP connect scan traffic shows the full SYN → SYN/ACK → ACK → RST/ACK pattern; TCP SYN scan traffic shows only SYN → SYN/ACK → RST (no ACK ever sent to complete the handshake).

```bash
nmap -sS 10.10.105.229
```

## UDP Scan (`-sU`)
```bash
sudo nmap -sU TARGET
```
UDP is a **connectionless** protocol — it does not require a handshake for connection establishment, so Nmap cannot guarantee a service listening on a UDP port will respond to a probe packet.

- Sending a UDP packet to an **open** port: no response is expected — this tells us nothing directly.
- Sending a UDP packet to a **closed** port: triggers an **ICMP port-unreachable** error (Type 3, Code 3) — this confirms the port is closed.
- Nmap infers a port is **open** when it sends a UDP probe and receives **no response at all** (since no ICMP unreachable came back).

Can be combined with a TCP scan type in the same command.

```bash
nmap -sU --top-ports 10 10.113.152.172
```
Example output:
```
PORT     STATE  SERVICE
53/udp   open   domain
67/udp   closed dhcps
123/udp  closed ntp
135/udp  closed msrpc
137/udp  closed netbios-ns
138/udp  closed netbios-dgm
161/udp  closed snmp
445/udp  closed microsoft-ds
631/udp  closed ipp
1434/udp closed ms-sql-m

Nmap done: 1 IP address (1 host up) scanned in 1.45 seconds
```
In Wireshark, closed UDP ports generate visible ICMP "Destination unreachable (Port unreachable)" replies for every closed port probed.

## Fine-Tuning Scope and Performance

### Specifying Ports
| Syntax | Effect |
|---|---|
| `-p22,80,443` | Scan a specific port list (22, 80, 443) |
| `-p1-1023` | Scan a port range (1 to 1023 inclusive) |
| `-p20-25` | Scan ports 20 through 25 inclusive |
| `-p-` | Scan all 65535 ports |
| `-F` | Scan the 100 most common ports |
| `--top-ports 10` | Scan the 10 most common ports |
| `-p5000-5500 TARGET_IP` | Scan all TCP ports between 5000 and 5500 |

### Scan Timing
```bash
nmap -T<0-5> TARGET
```
Six timing templates, from slowest/stealthiest to fastest/most aggressive:
| Template | Name |
|---|---|
| `-T0` | Paranoid |
| `-T1` | Sneaky |
| `-T2` | Polite |
| `-T3` | Normal (default if unspecified) |
| `-T4` | Aggressive |
| `-T5` | Insane |

- `-T0` scans one port at a time and waits 5 minutes between probes — useful for avoiding IDS alerts, but you can only guess how long a scan will take.
- `-T5` is fastest but can affect accuracy due to increased packet loss.
- `-T4` is often used during CTFs and practice-target learning.
- `-T1` is often used during real engagements where stealth matters more than speed.

### Rate and Parallelism Control
```bash
--min-rate <number>      # e.g. --min-rate 15 → rate >= 15 packets/sec
--max-rate <number>      # e.g. --max-rate 10 or --max-rate=10 → rate <= 10 packets/sec
--min-parallelism <numprobes>   # e.g. --min-parallelism=512 → maintain at least 512 probes in parallel
--max-parallelism <numprobes>
```
- `--min-rate` / `--max-rate` control the packet rate directly — e.g. `--max-rate 10` ensures the scanner sends no more than 10 packets per second.
- `--min-parallelism` / `--max-parallelism` control probing parallelisation — Nmap probes targets to discover which hosts are live and which ports are open; the parallelism parameter specifies how many such probes can run concurrently.

## Full Command Reference

| Port Scan Type | Example Command |
|---|---|
| TCP Connect Scan | `nmap -sT 10.113.152.172` |
| TCP SYN Scan | `sudo nmap -sS 10.113.152.172` |
| UDP Scan | `sudo nmap -sU 10.113.152.172` |

| Option | Purpose |
|---|---|
| `-p-` | Scan all ports |
| `-p1-1023` | Scan ports 1 to 1023 |
| `-F` | Scan the 100 most common ports |
| `-r` | Scan ports in consecutive order |
| `-T<0-5>` | Timing template — T0 slowest, T5 fastest |
| `--max-rate 50` | Cap rate at ≤ 50 packets/sec |
| `--min-rate 15` | Floor rate at ≥ 15 packets/sec |
| `--min-parallelism 100` | Maintain at least 100 probes in parallel |

## Key Takeaways
- **TCP connect scan (`-sT`)** completes the full 3-way handshake — works for unprivileged users, but is slower and more easily logged.
- **TCP SYN scan (`-sS`)** is the privileged default — faster, quieter (no full connection), same results as connect scan.
- **UDP scan (`-sU`)** relies on the *absence* of an ICMP port-unreachable response to infer an open port — inherently less reliable/slower than TCP scanning, since UDP has no handshake to confirm state directly.
- Scope (`-p`, `-F`, `--top-ports`) and performance (`-T`, `--min-rate`/`--max-rate`, `--min-parallelism`/`--max-parallelism`) options let you balance thoroughness, speed, and stealth depending on context (CTF vs real engagement).
- Next room in the series: **Nmap Advanced Port Scans**.
