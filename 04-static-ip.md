# 04 — Static IP Configuration

**Date:** May 14, 2026  
**Status:** Complete

---

## Overview

Configured a static private IP address on the server so it always has the same local network address (`192.168.1.100`) instead of receiving a random one from the router via DHCP.

---

## Background

**Dynamic IP (DHCP):** By default, routers assign IP addresses automatically via DHCP. The address can change each time the device connects or reboots. Fine for regular devices, problematic for servers.

**Static IP:** A manually configured IP address that never changes. Essential for servers because other devices need to always know where to find them — especially important for SSH access and DNS (Pi-hole).

**Public static IP vs Private static IP:**
- **Public static IP** — assigned by your ISP, costs money, never changes on the internet
- **Private static IP** — set locally within your own network, completely free, what we configured here

> **Why this matters for Pi-hole:** Pi-hole acts as a DNS server. Devices point to a specific IP to send DNS queries. If that IP changes, DNS breaks. Static IP is a prerequisite for running any server service.

---

## Steps

### 1. Check current network interfaces

```bash
ip a
```

**Findings:**
- `lo` — loopback interface, ignore this (it's just the server talking to itself)
- `eno1` — ethernet port, `state DOWN` (no cable plugged in)
- `wlo1` — WiFi interface, `state UP`, current IP `192.168.1.16/24` (dynamic)

> **Note:** Server is running on WiFi because the router is in another room. Ethernet is always preferred for servers (more stable, lower latency, no interference), but WiFi works fine for a homelab learning environment.

### 2. Locate the Netplan config file

Ubuntu manages network configuration through **Netplan** — a tool that uses YAML files to define network settings.

```bash
ls /etc/netplan/
```

Found: `50-cloud-init.yaml`

### 3. View existing config

```bash
sudo cat /etc/netplan/50-cloud-init.yaml
```

Confirmed `dhcp4: true` was set on `wlo1` — the router was assigning the IP dynamically.

### 4. Edit the config

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

> **Critical:** YAML is extremely strict about indentation. Use spaces only — never tabs. Wrong indentation will break networking entirely.

**Issue:** Initially only changed `dhcp4: true` to `dhcp4: false` but forgot to add the `addresses`, `routes`, and `nameservers` lines  
**Cause:** Missed adding the static IP details after disabling DHCP  
**Fix:** Used arrow keys and Enter/spacebar in nano to add the missing lines at the correct indentation level

Final config:

```yaml
network:
  version: 2
  wifis:
    wlo1:
      dhcp4: false
      addresses: [192.168.1.100/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
      access-points:
        "SpectrumSetup-95":
          auth:
            key-management: "psk"
            password: "[redacted]"
```

**Config breakdown:**

| Setting | Value | Meaning |
|---|---|---|
| `dhcp4: false` | false | Stop router from assigning IP dynamically |
| `addresses` | 192.168.1.100/24 | Static IP — /24 means subnet mask 255.255.255.0 |
| `via` | 192.168.1.1 | Default gateway (router) |
| `nameservers` | 8.8.8.8, 1.1.1.1 | Google and Cloudflare DNS (temporary — later replaced by Pi-hole) |

### 5. Apply changes

```bash
sudo netplan apply
```

**Issue:** SSH session dropped immediately  
**Cause:** Expected behavior — the server's IP changed from `192.168.1.16` to `192.168.1.100`, breaking the existing connection  
**Fix:** Reconnected using the new static IP:
```bash
ssh jjcrazy@192.168.1.100
```

**Issue:** Mac prompted "The authenticity of host can't be established"  
**Cause:** Mac stores known SSH hosts in `~/.ssh/known_hosts` by IP. The new IP `192.168.1.100` wasn't in the list yet  
**Fix:** Typed `yes` to accept — normal security prompt when connecting to a new IP for the first time

### 6. Verify

```bash
ip a
```

Confirmed `wlo1` now shows `192.168.1.100/24` instead of the old dynamic address.

---

## Key Concepts

- **DHCP** — Dynamic Host Configuration Protocol, automatically assigns IPs to devices on a network
- **Static IP** — manually configured, never changes
- **Subnetting** — `/24` notation means the first 24 bits are the network portion; supports 254 usable host addresses
- **Default gateway** — the router's IP, where traffic goes when it's destined for outside the local network
- **DNS servers** — where the device sends domain name lookup requests
- **Netplan** — Ubuntu's network configuration tool using YAML syntax
- **YAML indentation** — YAML uses indentation to define structure; spacing errors break the config

---

## Result

- Server permanently assigned IP `192.168.1.100` on the local network
- No longer dependent on router DHCP assignment
- SSH reconnection target is now always `192.168.1.100`
- Static IP confirmed as prerequisite for Pi-hole setup
