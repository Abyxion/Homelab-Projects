# homelab-docs

Personal homelab documentation — a running record of everything I've built, configured, and learned on my Ubuntu home server.

## About This Project

This server is an old HP laptop running Ubuntu, repurposed into a homelab for hands-on learning in networking, security, and IT. All projects are tailored toward real-world skills and certification prep (CompTIA Network+ and Security+).

## Server Info

| Item | Value |
|---|---|
| OS | Ubuntu (Noble) |
| Hostname | homelabserver |
| Static IP | 192.168.1.100 |
| Access | SSH via key-based authentication |

---

## Documentation Index

| # | Topic | Description |
|---|---|---|
| 01 | [SSH Hardening](01-ssh-hardening.md) | Securing remote access with key-based auth and explicit config |
| 02 | [Fail2ban](02-fail2ban.md) | Automated intrusion prevention for SSH |
| 03 | [UFW Firewall](03-ufw-firewall.md) | Host firewall setup and port management |
| 04 | [Static IP](04-static-ip.md) | Configuring a static private IP using Netplan |
| 05 | [Pi-hole](05-pihole.md) | Network-wide DNS sinkhole for ad and tracking blocking |

---

## Certifications in Progress
- CompTIA A+ ✅
- CompTIA Network+ (in progress)
- CompTIA Security+ (in progress)

---

## Future Projects
- VPN setup
- KVM/QEMU virtualization with Windows Server 2022 VM
- Active Directory configuration
- osTicket help desk deployment
- HTTPS for Pi-hole dashboard
- Port forwarding and NAT/PAT practice
- Networking tools: `nmap`, `tcpdump`, `netstat`, `ss`
