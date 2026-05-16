# 03 — UFW Firewall

**Date:** May 14, 2026  
**Status:** Complete (rules updated May 15, 2026 for Pi-hole)

---

## Overview

Enabled and configured UFW (Uncomplicated Firewall), Ubuntu's built-in host firewall, to control what network traffic is allowed to reach the server.

---

## Background

A firewall controls incoming and outgoing network traffic based on rules. UFW is a **host firewall** — it lives directly on the machine it protects and filters traffic at the OS level.

**Host firewall vs Network firewall:**
- **Host firewall (UFW)** — protects one specific device. Examples: UFW on Ubuntu, Windows Defender Firewall.
- **Network firewall** — protects the entire network, sits at the edge. Examples: home router, pfSense, Cisco ASA.

This server has both — the router acts as a network firewall, UFW acts as a host firewall. This is a real-world example of **defense in depth**.

**Key principle:** Only open ports that are actively needed. Every open port is a potential attack surface.

---

## Steps

### 1. Check UFW status

```bash
sudo ufw status
```

Output: `inactive` — UFW was installed by default on Ubuntu but not enabled.

### 2. Allow SSH before enabling

**Critical step:** Always allow SSH before enabling UFW. If UFW activates with no rules, it blocks everything — including your SSH connection, locking you out of the server.

```bash
sudo ufw allow 22/tcp
```

### 3. Enable UFW

```bash
sudo ufw enable
```

Prompted with a warning that existing connections may be disrupted. Typed `y` to confirm.

### 4. Verify

```bash
sudo ufw status
```

Output: `active` with port 22/tcp allowed.

---

## Port Rules Added Over Time

Rules were added as new services were installed. Each time a new service needed to be reachable, its port was opened explicitly.

### May 15, 2026 — Pi-hole ports added

**Issue:** Pi-hole web dashboard not loading — `ERR_CONNECTION_TIMED_OUT`  
**Cause:** Port 80 (HTTP) was blocked by UFW  
**Fix:**
```bash
sudo ufw allow 80/tcp
```

**Issue:** DNS queries not reaching Pi-hole — nslookup timing out  
**Cause:** Port 53 (DNS) was blocked by UFW  
**Fix:**
```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

> **Why both TCP and UDP for port 53?** DNS uses UDP for standard queries (faster, lower overhead) and TCP for larger responses. Both are required for full DNS functionality.

---

## Current UFW Rules

```bash
sudo ufw status
```

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH remote access |
| 80 | TCP | Pi-hole web dashboard (HTTP) |
| 53 | TCP | DNS queries |
| 53 | UDP | DNS queries |

---

## Key Concepts

- **UFW** — Uncomplicated Firewall, Ubuntu's default host firewall tool
- **Port** — a numbered endpoint for a specific type of network traffic
- **TCP vs UDP** — TCP is connection-based and reliable; UDP is faster but connectionless. Different services use different protocols.
- **Least privilege** — only open what's needed. Unnecessary open ports increase attack surface.
- **Port 22** — SSH | **Port 53** — DNS | **Port 80** — HTTP — all Network+ and Security+ exam staples
- **Defense in depth** — UFW is one layer; Fail2ban and SSH hardening add additional layers

---

## Result

- UFW active and enforcing rules
- Only necessary ports open
- All other traffic blocked by default
