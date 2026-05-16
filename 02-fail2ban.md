# 02 — Fail2ban

**Date:** May 14, 2026  
**Status:** Complete

---

## Overview

Installed and configured Fail2ban to automatically ban IP addresses that repeatedly fail SSH login attempts. This adds an automated intrusion prevention layer on top of SSH hardening.

---

## Background

Even with key-based SSH auth enabled, someone could still attempt to connect repeatedly. Fail2ban watches log files for patterns of failed login attempts and responds by creating temporary firewall rules to block the offending IP. This is a real-world example of automated intrusion prevention — a core Security+ concept.

**Analogy:** If UFW is the bouncer at the door, Fail2ban is the bouncer that remembers faces and bans anyone who tries to get in too many times.

---

## Steps

### 1. Install Fail2ban

```bash
sudo apt install fail2ban -y
```

### 2. Create a local config file

Fail2ban uses two config files — `jail.conf` (default) and `jail.local` (overrides). Changes should always go in `jail.local` so they aren't overwritten when Fail2ban updates.

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

### 3. Configure the SSH jail

```bash
sudo nano /etc/fail2ban/jail.local
```

Used `Ctrl+W` to search for `[sshd]` and configured:

```
enabled = true
port    = 22
maxretry = 3
bantime  = 1h
findtime = 10m
```

**Settings explained:**
- `enabled = true` — activates this jail
- `port = 22` — watching SSH port
- `maxretry = 3` — ban after 3 failed attempts
- `bantime = 1h` — ban lasts 1 hour
- `findtime = 10m` — 3 failures must occur within 10 minutes to trigger a ban

> **Note:** Some of these lines were commented out with `#`. Removed the `#` from each to activate them — same concept as the SSH config.

### 4. Enable and start Fail2ban

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

- `enable` — makes Fail2ban start automatically on boot
- `start` — starts it immediately

### 5. Verify

```bash
sudo systemctl status fail2ban
```

Confirmed: `active (running)` and `enabled`

---

## Key Concepts

- **Intrusion prevention** — automatically responding to threats without manual intervention
- **Log analysis** — Fail2ban works by reading system logs and detecting patterns
- **Defense in depth** — Fail2ban works alongside UFW and SSH hardening as another layer of protection
- **Brute force attack** — repeatedly trying passwords or keys to gain access; what Fail2ban defends against

---

## Result

- Fail2ban active and monitoring SSH login attempts
- Any IP with 3 failed attempts within 10 minutes is automatically banned for 1 hour
- Starts automatically on server reboot
