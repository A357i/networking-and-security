# Nmap advanced port scans
**Source:** TryHackMe — Nmap Advanced Port Scans
**Completed:** 2026-09-09
**Status:** done

## What this is for
Advanced Nmap covers the Advanced Port Scan (Null FIN Xmas scans , Evasion and Spoofing techniques) what i learnt from this room is techniques of working with nmap and --reason for communications between systems / to map firewall rules.

## Mental model
- **Null / FIN / Xmas scans** — set 0, 1, and 3 TCP flags respectively (never more than one packet per port — the difference is which flags are lit, not packet count). All rely on the same trick: RFC 793 says a closed port must reply RST to an unexpected packet, but an *open* port with no SYN present just stays silent. Silence gets read as "open|filtered," which is inherently ambiguous — you can't fully tell open from filtered from these three.
- **Maimon scan (`-sM`)** — sets FIN+ACK. Historically exploited a BSD quirk where certain systems dropped the packet on open ports. Mostly dead against modern systems (everything just RSTs), but useful to understand the general technique.
- **ACK scan (`-sA`)** — never tells you open vs closed. It tells you *unfiltered* vs *filtered*, i.e. whether a firewall is in the way at all. This is a firewall-mapping tool, not a port-discovery tool.
- **Window scan (`-sW`)** — same packets as ACK, but reads the TCP Window field in the RST reply. On some systems this leaks open vs closed even though ACK alone can't. Only useful when the target's stack happens to expose that quirk.
- **Custom scan (`--scanflags`)** — lets you set any flag combination yourself once you understand why the built-in scans use the flags they do.
- **Spoofing (`-S`) / decoys (`-D`)** — spoofing only works if you can also sniff the reply traffic (rare in practice); decoys don't hide you, they bury your real IP among several plausible ones in the target's logs.
- **Fragmentation (`-f` / `-ff`)** — splits the packet's data into 8- or 16-byte IP fragments to slip past firewalls/IDS that don't reassemble before inspecting.
- **Idle/zombie scan (`-sI`)** — uses a third idle host's predictable IP ID increments as a side channel, so the scan appears to come from the zombie, not you. Requires a genuinely idle, reachable host — a busy or offline "zombie" just gives useless results.

>

## Commands I actually ran
```bash
nmap -sN 10.113.148.69  # Null Scan - A TCP packet with no flags set will not trigger any response when it reaches an open port.
nmap -sF 10.113.148.69  # FIN Scan - Sends a packet with FIN flag set. Cannot be sure whether the port is open or a firewall is blocking traffic.
nmap -sX 10.113.148.69  # Xmas Scan - Sends packet with FIN, PSH and URG flags simultaneously. The same as the other two , in case the port is closed receives RST packet.
nmap -sM 10.113.148.69  # Maimon Scan - FIN/ACK flag. Most systems respond with RST packet but we won't be able to discover the open ports.
nmap -sA 10.113.148.69  # TCP ACK Scan - TCP packet with ACK Flag. Target respomds with RST regardless the status of the port. This type of scan is better to discover firewall rule sets and configs and you will learn which ports were not blocked by firewall.
nmap -sW 10.113.148.69  # Window Scan - Examines the TCP Window field of the RST packets returned which on specifis systems can reveal that the port is open.
nmap --scanflags RST -sT 10.113.148.69
nmap --scanflags RST 10.113.148.69  # Custom Scan - Experiment with your own flag combination.
nmap -S SPOOFED_IP 10.113.148.69
nmap -D 10.10.0.1,10.10.0.2,ME 10.113.148.69    # Decoy Scan - will make the scan appear as comming from different ip address. 3rd order
nmap -D 10.10.0.1,10.10.0.2,RND,RND,ME 10.113.148.69    # random 3rd 4th 5th one is attacker's ip address.
sudo nmap -sS -p80 -f 10.113.148.69   # SYN on 80, fragmented (-f = 8-byte fragments). The new part is -f, not -sS.
nmap -sI ZOMBIE_IP 10.113.148.69      # placeholder, failed to resolve
nmap -sI 10.10.5.5 10.113.148.69      # real IP, zombie unreachable/no probes returned
nmap -sS 10.113.148.69
nmap -sS --reason 10.113.148.69       # Nmap provides reasoning
nmap -sS -vv 10.113.148.69
```
## Keep
| Flag | Use |
|---|---|
| `-sN -sF -sX` | RFC 793 odd-flag scans; result is often open\|filtered |
| `-sA` | firewall map, not port map |
| `-D ...,ME` | decoys; ME is your real IP in that list |
| `-f` | fragment |
| `--reason` | why Nmap chose that state |

## Open
- [ ] when would I pick ACK over SYN in a real engagement
- [SYN first, to find listeners. ACK only after that, when I need to see how the firewall treats traffic that is not a connection attempt.]

## Room questions i had ?
1. What is a stateless firewall ? 
Stateless firewall: A firewall that judges each packet completely on its own, with no memory of what came before it. In simpler words, a filter, If a packet is destined for port 80 and matches the firewall's rules, it is allowed through. The firewall does not remember whether that packet belongs to an already established TCP connection—it only evaluates the packet itself. This is Stateless.
2. What are TCP flags ?
TCP flags are bits in the TCP header that indicate the purpose of a packet. They tell the receiving device whether the packet is trying to start a connection (SYN), acknowledge data (ACK), close a connection (FIN), reset a connection (RST), or perform another TCP function. / TCP flags are bits in the TCP header that tell the receiving device what type of packet it is and how it should handle it.
