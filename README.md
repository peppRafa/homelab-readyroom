# homelab-readyroom
The Captain's Ready Room - Network, infrastructure and security engineering homelab. Documented as built.

# 🛡️ The Captain's Ready Room — Homelab Engineering Journal

> A production-grade homelab built for learning and applying network engineering,
> infrastructure automation, and security operations. Documented as built — 
> including what broke and why.

---

## Architecture

| Layer | Technology |
|---|---|
| Edge Router | OpenWrt x86 |
| Virtualization | Proxmox VE |
| DNS / Ad-blocking | AdGuard Home |
| VPN | WireGuard |
| Observability | Prometheus + Grafana + Loki |
| Log shipping | Promtail + rsyslog |
| Alerting | ntfy (self-hosted) |
| Containers | Docker / Docker Compose |

---

## Network Topology

ISP
└── OpenWrt Mini PC (192.168.10.1) — edge router, DHCP, dnsmasq, WireGuard
├── Proxmox I3 Laptop — hypervisor
│    ├── Services LXC — Docker stack
│    └── AdGuard LXC — DNS with DoH
├── NAS Core2Quad
└── I5 Daily Driver

---

## What's in here

| Folder | Contents |
|---|---|
| `/architecture` | Topology diagrams, design decisions |
| `/infrastructure` | Configs — Prometheus, Grafana, Docker, OpenWrt |
| `/runbooks` | Step-by-step operational procedures |
| `/incident-logs` | Real debugging sessions — not sanitized tutorials |
| `/configs` | Sanitized service configs |

---

## Incident Logs

Real problems, real fixes. This is where the learning actually happens.

| Date | Incident | Root Cause |
|---|---|---|
| 2026-05-13 | Intermittent WAN drops on all devices | odhcpd broadcasting IPv6 RA lifetime=0 — ISP has no IPv6 uplink |
| 2026-05-11 | Monitoring stack rebuild | Promtail stale labels + rsyslog missing on Debian 12 |

---

## Roadmap

- [x] WireGuard VPN (split tunnel, 2 peers)
- [x] AdGuard Home with DoH upstreams
- [x] Prometheus + Grafana + Loki observability stack
- [x] Self-hosted ntfy alerting (10 alerts)
- [ ] Mosh on router and servers
- [ ] ntopng traffic analysis
- [ ] Reverse proxy + HTTPS (Caddy)
- [ ] VLAN segmentation (LAN / IoT / Management / DMZ)
- [ ] Wazuh / Suricata security tooling

---

## ⚠️ Security note

All configs in this repo are sanitized — tokens, private keys, and internal IPs
have been removed or replaced with placeholders. Never commit real credentials.
