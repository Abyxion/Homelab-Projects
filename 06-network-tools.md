# 06 — Network Analysis & Reconnaissance

**Date:** May 16–17, 2026  
**Status:** Complete

-----

## Overview

Explored five core networking tools used for network analysis, traffic monitoring, port auditing, and network discovery. These tools are used daily by IT professionals and security analysts, and are heavily tested on CompTIA Network+ and Security+.

-----

## Tools Covered

|Tool     |Purpose                                           |
|---------|--------------------------------------------------|
|`ip`     |Network interface and routing information         |
|`ss`     |Socket statistics — open ports and connections    |
|`netstat`|Network statistics — older alternative to ss      |
|`tcpdump`|Live packet capture and traffic analysis          |
|`nmap`   |Network scanner — host discovery and port scanning|

-----

## Tool 1 — ip

The `ip` command shows network interface information and routing tables.

### Check network interfaces

```bash
ip a
```

**Output breakdown:**

- `lo` — loopback interface, the server talking to itself (`127.0.0.1`)
- `eno1` — ethernet port, `state DOWN` (no cable connected)
- `wlo1` — WiFi interface, `state UP`, static IP `192.168.1.100/24`

### Check routing table

```bash
ip route
```

**Output:**

```
default via 192.168.1.1 dev wlo1 proto static
192.168.1.0/24 dev wlo1 proto kernel scope link src 192.168.1.100
```

**Breakdown:**

- **Line 1** — default route. Any traffic with no specific destination gets sent to the router (`192.168.1.1`). This is how the server reaches the internet. `proto static` means we set this manually in Netplan.
- **Line 2** — local network route. Traffic destined for anything on `192.168.1.0/24` (the home network) is handled directly without going through the router.

**Connection to previous work:** These routes were created when we configured the static IP in Netplan. The `via: 192.168.1.1` and `addresses: [192.168.1.100/24]` lines in the YAML file directly created these routing table entries.

-----

## Tool 2 — ss

`ss` (socket statistics) shows all open ports and active connections on the server.

```bash
ss -tuln
```

**Flags:**

- `-t` — show TCP
- `-u` — show UDP
- `-l` — show listening ports only
- `-n` — show port numbers instead of service names

**Output breakdown:**

|Column            |Meaning                                  |
|------------------|-----------------------------------------|
|Netid             |Protocol (TCP or UDP)                    |
|State             |LISTEN = waiting for connections         |
|Local Address:Port|What’s open on this server               |
|Peer Address:Port |Who’s connected (empty if just listening)|

**Address notation:**

- `0.0.0.0` — IPv4, listening on all interfaces (reachable from network)
- `[::]` — IPv6, listening on all interfaces (same idea, different protocol)
- `127.0.0.1` — only the server itself can reach this port

**Ports visible on this server:**

|Port|Protocol|Service                               |
|----|--------|--------------------------------------|
|22  |TCP     |SSH                                   |
|53  |TCP/UDP |DNS (Pi-hole)                         |
|80  |TCP     |HTTP (Pi-hole dashboard)              |
|443 |TCP     |HTTPS (Pi-hole)                       |
|631 |TCP     |CUPS printing service (localhost only)|

**Security relevance:** `ss` is used to audit what a server is exposing. Any unrecognized port listening on `0.0.0.0` is a red flag — it could indicate malware or an unauthorized service.

-----

## Tool 3 — netstat

`netstat` is an older tool that provides similar output to `ss`. Still widely used and appears on certification exams.

```bash
netstat -tuln
```

**Key differences from ss:**

- Uses `Proto` instead of `Netid`
- Lists `tcp6` and `udp6` separately instead of using bracket notation
- Includes a `Foreign Address` column showing active connections (shows `0.0.0.0:*` when just listening)

**Takeaway:** `ss` and `netstat` do the same job. `ss` is the modern replacement. Know both for exams and real world use.

-----

## Tool 4 — tcpdump

`tcpdump` captures live network traffic in real time. Used for troubleshooting, security monitoring, and verifying services are working correctly.

### Basic capture on WiFi interface

```bash
sudo tcpdump -i wlo1
```

Captures all traffic — outputs rapidly, use Ctrl+C to stop.

### Filtered capture — DNS traffic only

```bash
sudo tcpdump -i wlo1 port 53
```

**What we saw:** Real-time DNS queries from the Mac (`192.168.1.62`) being sent to Pi-hole (`192.168.1.100`) and responses coming back. Domains like `claude.ai`, `google.com`, and Apple services were visible in the traffic.

**Example output breakdown:**

```
192.168.1.62 > 192.168.1.100.domain: AAAA? claude.ai
```

- `192.168.1.62` — Mac sending a DNS query
- `>` — direction of traffic
- `192.168.1.100` — Pi-hole receiving the query
- `AAAA? claude.ai` — asking for the IPv6 address of claude.ai

```
192.168.1.100 > 192.168.1.62: A 160.79.104.10
```

- Pi-hole responding with the IPv4 address for claude.ai

**Troubleshooting use case:** If a service isn’t working, run tcpdump and watch the traffic. If packets arrive but no response comes back, the problem is the service or firewall. If no packets arrive at all, the problem is earlier in the network path.

-----

## Tool 5 — nmap

`nmap` is a network scanner used to discover hosts and open ports. Used by network administrators for inventory and by security professionals for penetration testing.

> **Important:** Only scan networks you own or have explicit permission to scan. Unauthorized scanning is illegal.

### Install nmap

```bash
sudo apt install nmap -y
```

### Scan a single host

```bash
nmap 192.168.1.100
```

**Output:**

```
PORT     STATE  SERVICE
22/tcp   open   ssh
53/tcp   open   domain
80/tcp   open   http
443/tcp  open   https
```

Confirmed only the ports we intentionally opened are visible. 996 other ports showed as closed — exactly what a hardened server should look like.

### Scan entire network

```bash
nmap 192.168.1.0/24
```

**Devices discovered on the home network:**

|IP           |Hostname      |Open Ports     |Device                   |
|-------------|--------------|---------------|-------------------------|
|192.168.1.1  |SAX1V1S.lan   |53, 80, 443    |Spectrum router          |
|192.168.1.62 |—             |5000, 7000     |MacBook (AirPlay/AirDrop)|
|192.168.1.100|homelabserver |22, 53, 80, 443|Homelab server           |
|192.168.1.106|iPhone-366.lan|49152, 62078   |iPhone                   |

**What this demonstrates:** nmap mapped the entire home network in seconds, showing every online device and its exposed ports. This is called **network discovery** — a fundamental skill in both network administration and security.

-----

## Bonus — UFW Rule Order Lesson

During the tcpdump session we attempted to simulate a firewall failure by running:

```bash
sudo ufw deny 53
```

**Issue:** Traffic continued flowing through port 53 despite adding the deny rule.

**Cause:** UFW processes rules **top to bottom** and stops at the first match. The existing `ALLOW 53/tcp` and `ALLOW 53/udp` rules were above the new `DENY 53` rule, so traffic matched the allow rules first and the deny rule was never reached.

**Key concept:** Firewall rule order matters. In any firewall system (UFW, Cisco ACLs, AWS Security Groups), rules are evaluated in order. A deny rule placed below an allow rule for the same port will be ignored.

**Fix:** To properly block a port that has an existing allow rule, either delete the allow rule first or insert the deny rule above it.

**Cleanup:** Removed the ineffective deny rules using:

```bash
sudo ufw status numbered
sudo ufw delete [rule number]
```

-----

## Key Concepts Covered

|Concept              |Tool       |Exam Relevance|
|---------------------|-----------|--------------|
|Network interfaces   |ip a       |Network+      |
|Routing table        |ip route   |Network+      |
|Default gateway      |ip route   |Network+      |
|Port auditing        |ss, netstat|Security+     |
|IPv4 vs IPv6 notation|ss, netstat|Network+      |
|Packet capture       |tcpdump    |Security+     |
|DNS traffic flow     |tcpdump    |Network+      |
|Network discovery    |nmap       |Security+     |
|Firewall rule order  |UFW/nmap   |Security+     |
|Attack surface       |nmap       |Security+     |

-----

## Future Projects

- [ ] Set up Kali Linux as a VM on gaming laptop
- [ ] Use Kali to perform controlled penetration testing against the homelab server
- [ ] Practice with Wireshark for visual packet analysis
- [ ] Explore Metasploit for offensive security learning