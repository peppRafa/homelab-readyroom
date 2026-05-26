# Wake-on-LAN Setup

**Last updated:** 2026-05-26  
**Author:** Rafael Peppi

---

## Overview

Wake-on-LAN (WoL) allows powering on machines remotely via a magic packet
sent over the network. Combined with BIOS power restore settings, this ensures
the homelab recovers automatically after power outages.

---

## Homelab WoL Status

|      Machine     |      IP       |      WoL        |   Power Restore   |         Notes               |
|------------------|---------------|-----------------|-------------------|-----------------------------|
| OpenWrt Mini PC  | 192.168.10.1  |       N/A       |  ✓ Power On       | Gateway — must boot first   |
| NAS Core2Quad    | 192.168.10.50 |   ✓ Enabled     |  ✓ Power On       | PCI/PCIe WoL in BIOS        |
| Proxmox Laptop   | 192.168.10.20 | ✗ Not supported |  ✗ No BIOS option | Lenovo InsydeH2O limitation |
| Services LXC 100 | 192.168.10.30 |       N/A       |  ✓ onboot=1       | Starts with Proxmox         |
| AdGuard LXC 101  | 192.168.10.40 |       N/A       |  ✓ onboot=1       | Starts with Proxmox        Zakhar Khrashchevskyi |

---

## BIOS Configuration

### NAS (Core2Quad — AMI BIOS)
`Power → APM Configuration`
- `Restore on AC Power Loss` → **Power On**
- `Power On By PCI Devices` → **Enabled**
- `Power On By PCIE Devices` → **Enabled**

### OpenWrt Mini PC (Aptio BIOS)
`Chipset → PCH-IO Configuration`
- `Restore on AC Power Loss` → **Power On**

### Proxmox Laptop (Lenovo — InsydeH2O)
- No power restore option available in BIOS
- Lid-close suspend disabled via systemd (already configured)

---

## Sending a WoL Packet

### Install wakeonlan on Acer (one time)
```bash
sudo apt install wakeonlan
```

### Get NAS MAC address
```bash
ssh root@192.168.10.50 "ip link show | grep -A1 enp"
```

### Wake the NAS
```bash
wakeonlan <NAS-MAC-ADDRESS>
```

### Verify it booted
```bash
ping 192.168.10.50
# then
ssh root@192.168.10.50
```

---

## Clean NAS Shutdown (before WoL)
```bash
ssh root@192.168.10.50 "shutdown -h now"
```

Wait ~30 seconds for full power down before sending WoL packet.

---

## Notes

- WoL only works on the local LAN — magic packet doesn't cross the router by default
- Remote WoL (over WireGuard) requires additional router configuration — not yet implemented
- Proxmox laptop has no WoL or power restore — manual intervention needed after power outage
