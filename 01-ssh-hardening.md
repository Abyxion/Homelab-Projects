# 01 — SSH Hardening

**Date:** May 14, 2026  
**Status:** Complete

---

## Overview

Secured remote SSH access by explicitly enforcing key-based authentication and disabling password-based login. Previously these settings existed in the config file but were commented out, meaning the system was relying on defaults rather than actively enforcing them.

---

## Background

SSH (Secure Shell) is the protocol used to remotely access the server from a Mac terminal. By default Ubuntu allows password-based SSH login, which is vulnerable to brute force attacks. Key-based authentication is more secure — only a device with the matching private key can connect.

---

## Steps

### 1. Check existing SSH config

```bash
sudo cat /etc/ssh/sshd_config | grep -E "Port|PasswordAuthentication|PubkeyAuthentication"
```

**Issue:** Output showed all three settings commented out with `#`:
```
#Port 22
#PasswordAuthentication no
#PubkeyAuthentication yes
```

**Cause:** In Linux config files, `#` marks a line as a comment — the system ignores it completely. These were placeholder reference lines, not active rules. The system was falling back to compiled-in defaults.

**Why this matters:** Relying on defaults is risky. If a future update changes the default behavior, your security settings silently change with it. Always explicitly set what you care about.

### 2. Edit the SSH config

```bash
sudo nano /etc/ssh/sshd_config
```

Used `Ctrl+W` to search for each setting. Removed the `#` from the following lines to uncomment and activate them:

```
Port 22
PasswordAuthentication no
PubkeyAuthentication yes
```

> **Note:** `KbdInteractiveAuthentication no` was already uncommented — this disables keyboard-interactive auth, another potential password-based login method. Left as-is.

> **Decision on port:** Chose to keep port 22 (default) for now. The server is local network only with no port forwarding through the router, so exposure to internet bots is not a concern. Port change will be revisited when a VPN is configured.

### 3. Save and restart SSH

Save with `Ctrl+X` → `Y` → `Enter`, then restart the SSH service:

```bash
sudo systemctl restart ssh
```

> `systemctl restart` runs silently on success — no output means it worked.

### 4. Verify

```bash
sudo systemctl status ssh
```

Confirmed: `active (running)`

---

## Key Concepts

- **SSH key-based authentication** — uses a public/private key pair instead of a password. The server holds the public key, the client holds the private key. No password is ever sent over the network.
- **Commenting in config files** — `#` at the start of a line disables it. Removing `#` activates it.
- **Defense in depth** — SSH hardening is one layer; Fail2ban and UFW add additional layers on top.
- **Port 22** — the default SSH port. A Network+ and Security+ exam staple.

---

## Result

- Password authentication explicitly disabled
- Key-based authentication explicitly enabled
- Server accessible via SSH from Mac without a password prompt (key handles authentication automatically)
