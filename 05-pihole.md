# 05 — Pi-hole

**Date:** May 15, 2026  
**Status:** Complete

---

## Overview

Installed and configured Pi-hole as a network-wide DNS sinkhole. All DNS requests from devices pointed at Pi-hole are filtered against a blocklist of 82,208 known ad and tracking domains before being resolved.

---

## Background

**How DNS works:** When you type a website address, your device asks a DNS server "what's the IP address for this domain?" The DNS server looks it up and responds. Your device then connects to that IP.

**What Pi-hole does:** Pi-hole sits between your devices and the internet as a DNS server. When a device asks for the IP of a known ad or tracking domain, Pi-hole checks its blocklist and returns nothing — the connection is never made. This happens before the page even loads, making it more effective than browser-based ad blockers.

**Why static IP is a prerequisite:** Devices are configured to send DNS queries to a specific IP address. If that IP changes, DNS breaks. The static IP configured in the previous step (`192.168.1.100`) was required before Pi-hole could be set up.

---

## Steps

### 1. Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

The installer displayed "Script called with non-root privileges" in orange — informational only, not an error. The installer handles sudo automatically.

**Interactive setup choices:**

| Setting | Choice | Reason |
|---|---|---|
| Upstream DNS | Cloudflare (DNSSEC) | Fast, privacy focused; DNSSEC verifies DNS responses haven't been tampered with |
| Query Logging | Enabled | Logs every DNS request for visibility and learning |
| Privacy Mode | Show Everything | Full client and domain visibility for homelab purposes |

**Installation complete screen confirmed:**
- Pi-hole running on `192.168.1.100` (static IP — prerequisite verified)
- Web dashboard available at `http://192.168.1.100/admin`
- Admin password auto-generated (saved separately)
- 82,208 domains loaded onto blocklist automatically

### 2. Open required firewall ports

**Issue:** Web dashboard returned `ERR_CONNECTION_TIMED_OUT` in browser  
**Cause:** UFW was blocking port 80 (HTTP) — only port 22 was open from the previous session  
**Fix:**
```bash
sudo ufw allow 80/tcp
```

Dashboard loaded successfully after adding the rule.

**Issue:** DNS queries not reaching Pi-hole — `nslookup` from Mac timed out  
**Cause:** UFW was blocking port 53 (DNS)  
**Fix:**
```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

> **Why both TCP and UDP?** DNS uses UDP for standard queries (faster) and TCP for larger responses. Both must be open for full DNS functionality.

### 3. Point Mac to Pi-hole as DNS server

**Mac:** System Settings → Network → Wi-Fi → Details → DNS

Configured DNS server order:
1. `192.168.1.100` — Pi-hole (primary)
2. `192.168.1.1` — router (backup)
3. `2603:8080:2e00:63b0::1` — ISP IPv6 (backup)

> **Production vs homelab:** In production you'd use Pi-hole as the only DNS server to ensure nothing bypasses it. In a homelab, keeping backup DNS servers prevents losing internet access if Pi-hole goes down.

**Issue:** No traffic showing in Pi-hole query log after configuring Mac DNS  
**Cause:** Brave browser has built-in DNS over HTTPS (DoH) that bypasses system DNS settings entirely  
**Fix:** Disabled in Brave → Settings → Privacy and Security → Security → Use Secure DNS → Off

**Verified Mac was using Pi-hole:**
```bash
scutil --dns | grep nameserver
```
Confirmed `192.168.1.100` listed as `nameserver[0]`.

**Verified Pi-hole was responding to DNS queries:**
```bash
nslookup google.com 192.168.1.100
```
Resolved successfully.

### 4. Verify Pi-hole is working

Accessed dashboard at `http://192.168.1.100/admin`

**Dashboard confirmed working:**
- Status: Active, 64 queries/minute
- Mac (`192.168.1.62`) showing 133+ queries logged
- Tracking domains blocked in real time (e.g. `metrics.icloud.com`)
- 82,208 domains on blocklist

**Query log breakdown:**

| Icon | Meaning |
|---|---|
| 🟢 Green database | Query allowed and resolved normally |
| 🔴 Red circle | Query blocked by Pi-hole — domain is on the blocklist |

**DNS record types visible in query log:**

| Type | Meaning |
|---|---|
| A | IPv4 address record |
| AAAA | IPv6 address record |
| HTTPS | Newer record type for HTTPS connections |

**Real-world blocking example:**  
`metrics.icloud.com` — Apple telemetry and usage data collection — automatically blocked. The connection was never made and Apple never received the data.

---

## Key Concepts

- **DNS sinkhole** — intercepts DNS queries and blocks requests to known bad domains before a connection is made
- **Port 53** — the standard DNS port (TCP and UDP)
- **Port 80** — HTTP, used for the Pi-hole web dashboard
- **DNS over HTTPS (DoH)** — encrypts DNS queries inside HTTPS, bypassing traditional DNS filtering. Browsers like Brave use this by default.
- **Upstream DNS** — the DNS provider Pi-hole forwards requests to after filtering (Cloudflare in this case)
- **DNSSEC** — DNS Security Extensions; verifies that DNS responses haven't been tampered with in transit
- **Network visibility** — Pi-hole's query log reveals constant background DNS traffic from devices including telemetry, app check-ins, and tracking — most of which users never see

---

## Future Improvements
- [ ] Add HTTPS to the Pi-hole dashboard using a self-signed certificate (HTTP is currently unencrypted)
- [ ] Change auto-generated admin password
- [ ] Point additional devices on the network to Pi-hole
- [ ] Explore additional blocklists

---

## Result

- Pi-hole active and processing 64+ queries per minute
- 82,208 known ad and tracking domains blocked network-wide
- Real-time query logging enabled for full network visibility
- Mac traffic confirmed routing through Pi-hole
