# 07 — WireGuard VPN

**Date:** May 22–23, 2026  
**Status:** Partially complete — pending port forwarding setup (awaiting dedicated router)

---

## Overview

Installed and configured WireGuard as a VPN server on the homelab. WireGuard is a modern, lightweight VPN protocol that creates an encrypted tunnel between devices. Once port forwarding is configured on a dedicated router, this will allow secure remote access to the home network from anywhere.

---

## Why WireGuard?

- Free and open source
- Lightweight and fast compared to older protocols like OpenVPN
- Built into the Linux kernel
- Simple configuration
- Industry standard for homelab and professional VPN setups

---

## How It Works

```
Outside Device → Public IP → Router → Port Forward 51820 → WireGuard Server → Home Network
```

Your server acts as the VPN gateway. Devices connect to it through an encrypted tunnel. Once connected, they can reach anything on your home network as if they were physically there.

**Why port forwarding is required:**
The router is the gatekeeper between your home network and the internet. It has one public IP address. Port forwarding tells the router "when traffic arrives on port 51820, send it to the WireGuard server at 192.168.1.100." Without this rule, outside traffic hits the router and gets dropped because the router doesn't know where to send it.

---

## Network Layout

| Device | IP | Role |
|---|---|---|
| Router | 192.168.1.1 | Network gateway |
| Homelab server | 192.168.1.100 | WireGuard VPN server |
| VPN network | 10.0.0.0/24 | Virtual VPN subnet |
| Server VPN IP | 10.0.0.1 | Server's address on VPN |
| Mac client VPN IP | 10.0.0.2 | Mac's address on VPN |

---

## Installation

### Install WireGuard
```bash
sudo apt install wireguard -y
```

**Note:** During installation a pending kernel upgrade was detected. Rebooted first to load the new kernel before proceeding.

```bash
sudo reboot
```

Verified kernel updated:
```bash
uname -r
```

---

## Key Generation

WireGuard uses public/private key pairs for authentication — no passwords. Each device needs its own key pair.

### Generate server keys
```bash
wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key
```

### Generate client keys (for Mac)
```bash
wg genkey | sudo tee /etc/wireguard/client_private.key
sudo cat /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key
```

**Key concepts:**
- **Private key** — stays on the device it belongs to, never shared
- **Public key** — shared with the other side, used to identify the device
- Server holds client's public key → knows who's allowed to connect
- Client holds server's public key → knows who it's connecting to

**Troubleshooting encountered:** First key generation attempt resulted in a backtick character at the start of the client public key, causing a "Key is not the correct length or format" error. Regenerated keys cleanly to fix this.

---

## Server Configuration

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = [SERVER_PRIVATE_KEY]

PostUp = iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o wlo1 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o wlo1 -j MASQUERADE

[Peer]
PublicKey = [CLIENT_PUBLIC_KEY]
AllowedIPs = 10.0.0.2/32
```

**Config breakdown:**
- `Address` — server's IP on the VPN network
- `ListenPort` — port WireGuard listens on (51820 is the standard WireGuard port)
- `PrivateKey` — server's private key
- `PostUp/PostDown` — iptables rules that enable NAT masquerading when WireGuard starts/stops, allowing VPN clients to route traffic through the server
- `[Peer]` — defines an allowed client
- `AllowedIPs` — the VPN IP assigned to this client

---

## Enable IP Forwarding

Required so the server can route VPN traffic to the internet.

```bash
sudo nano /etc/sysctl.conf
```

Uncommented:
```
net.ipv4.ip_forward=1
```

Applied:
```bash
sudo sysctl -p
```

Also updated UFW default forward policy:
```bash
sudo nano /etc/default/ufw
```

Changed:
```
DEFAULT_FORWARD_POLICY="DROP"
```
To:
```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

**Why this matters:** UFW's default was to DROP all forwarded/routed traffic. This blocked VPN traffic from passing through the server even though WireGuard was running. Changing to ACCEPT allows the server to forward traffic between the VPN tunnel and the local network.

---

## Open Firewall Port

```bash
sudo ufw allow 51820/udp
sudo ufw reload
```

---

## Start WireGuard

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show
```

Confirmed interface `wg0` running on port 51820 with peer registered.

---

## Mac Client Configuration

Installed WireGuard from the Mac App Store.

Created tunnel with this config:

```ini
[Interface]
PrivateKey = [CLIENT_PRIVATE_KEY]
Address = 10.0.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = [SERVER_PUBLIC_KEY]
AllowedIPs = 10.0.0.0/24, 192.168.1.0/24
Endpoint = 192.168.1.100:51820
```

**Config breakdown:**
- `Address` — Mac's IP on the VPN network
- `DNS` — using Cloudflare instead of VPN DNS to avoid resolution issues
- `AllowedIPs` — split tunneling: only route VPN and home network traffic through the tunnel, everything else goes normally. This allows normal internet use while connected to the VPN.
- `Endpoint` — server's IP and WireGuard port

**Split tunneling explained:** Instead of routing ALL traffic through the VPN (`0.0.0.0/0`), only traffic destined for the VPN subnet (`10.0.0.0/24`) and home network (`192.168.1.0/24`) goes through the tunnel. Everything else uses the normal internet connection.

---

## Troubleshooting

### Issue 1: Key format error
**Error:** `Key is not the correct length or format`  
**Cause:** Backtick character at the start of the key from terminal output  
**Fix:** Regenerated client keys cleanly

### Issue 2: VPN activated but all traffic blocked
**Cause:** `AllowedIPs = 0.0.0.0/0` was routing ALL traffic through the tunnel including the Claude session  
**Fix:** Changed to split tunneling `10.0.0.0/24, 192.168.1.0/24`

### Issue 3: SSH through VPN timing out
**Cause:** UFW default forward policy was DROP, blocking routed traffic  
**Fix:** Changed `DEFAULT_FORWARD_POLICY` to `ACCEPT` in `/etc/default/ufw`

### Issue 4: Handshake not completing on local network
**Cause:** Hairpin NAT issue — when client and server are on the same local network, VPN traffic can't loop back through the router properly  
**Status:** Expected behavior. VPN is configured correctly and will work when tested from outside the network.

---

## What's Needed to Complete Setup

1. **Port forwarding on router** — forward UDP port 51820 to `192.168.1.100`
2. **Update client endpoint** — change from local IP to public IP in Mac WireGuard config
3. **Test from outside network** — connect Mac to phone hotspot, activate VPN, verify handshake and SSH to `10.0.0.1`

**Current blocker:** No admin access to the shared Spectrum router. Will complete when dedicated router is available.

---

## Key Concepts Covered

| Concept | Relevance |
|---|---|
| VPN tunneling | Security+ |
| Public/private key cryptography | Security+ |
| Port forwarding | Network+ |
| NAT masquerading | Network+ |
| Split tunneling | Security+ |
| Hairpin NAT | Network+ |
| UFW forward policy | Security+ |
| IP forwarding | Network+ |
| WireGuard protocol | Industry standard |

---

## Future Steps
- [ ] Get dedicated router with admin access
- [ ] Set up port forwarding (UDP 51820 → 192.168.1.100)
- [ ] Test VPN from outside network
- [ ] Add phone as WireGuard client
- [ ] Set up port forwarding project (NAT/PAT concepts)
