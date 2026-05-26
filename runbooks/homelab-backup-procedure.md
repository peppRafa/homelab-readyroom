# Homelab Backup Procedure

**Last updated:** 2026-05-26  
**Author:** Rafael Peppi  
**Applies to:** OpenWrt router, Proxmox node, Docker stack

---

## Overview

Manual backup procedure for core homelab configuration files.  
Target: NAS at `192.168.10.50`, path `/mnt/storage/backups/homelab/`

No VM snapshots or data backups — config files only.  
Recovery goal: rebuild any node from scratch using these files.

---

## Pre-backup: NAS Health Check

```bash
# Verify all disks healthy
for disk in /dev/sd{a,b,c,d}; do
  echo "=== $disk ===" && smartctl -H $disk
done

# Sync SnapRAID parity before relying on it
snapraid sync
snapraid status


Expected: all SMART `PASSED`, sync completes with `Everything OK`.

---

## Directory Structure

```
/mnt/storage/backups/homelab/
├── openwrt/
│   ├── openwrt-backup-YYYYMMDD.tar.gz
│   ├── network-uci-YYYYMMDD.txt
│   └── packages-YYYYMMDD.txt
├── proxmox/
│   ├── interfaces
│   ├── lxc/
│   │   ├── 100.conf
│   │   └── 101.conf
│   └── node/
└── docker/
└── monitoring/
```
---

## 1. OpenWrt Backup

**On the router (`192.168.10.1`):**

```bash
sysupgrade --create-backup /tmp/openwrt-backup-$(date +%Y%m%d).tar.gz
uci export network > /tmp/network-uci-$(date +%Y%m%d).txt
apk list --installed > /tmp/packages-$(date +%Y%m%d).txt
```

**Pull to NAS (from NAS):**

```bash
scp root@192.168.10.1:/tmp/openwrt-backup-*.tar.gz /mnt/storage/backups/homelab/openwrt/
scp root@192.168.10.1:/tmp/network-uci-*.txt /mnt/storage/backups/homelab/openwrt/
scp root@192.168.10.1:/tmp/packages-*.txt /mnt/storage/backups/homelab/openwrt/
```

**What's covered:**
- Full `/etc/config/` including WireGuard keys (stored inline in UCI)
- Explicit network UCI export (human-readable)
- Installed package list for post-upgrade reinstall

---

## 2. Proxmox Backup

**From Proxmox node (`192.168.10.20`) to NAS:**

```bash
rsync -av /etc/pve/lxc/ root@192.168.10.50:/mnt/storage/backups/homelab/proxmox/lxc/
rsync -av /etc/pve/nodes/services/ root@192.168.10.50:/mnt/storage/backups/homelab/proxmox/node/
rsync -av /etc/network/interfaces root@192.168.10.50:/mnt/storage/backups/homelab/proxmox/
```

**What's covered:**
- LXC container configs (100.conf, 101.conf)
- Proxmox node configuration
- Host network interfaces

---

## 3. Docker Stack Backup

**From Services LXC (`192.168.10.30`) to NAS:**

```bash
rsync -av /root/monitoring/ root@192.168.10.50:/mnt/storage/backups/homelab/docker/monitoring/
```

**What's covered:**
- `docker-compose.yml`
- Prometheus, Grafana, Loki config files
- `.env` files (credentials — never commit to Git)

---

## Post-backup: Verify

```bash
# On NAS
ls -lh /mnt/storage/backups/homelab/openwrt/
ls -lh /mnt/storage/backups/homelab/proxmox/
ls -lh /mnt/storage/backups/homelab/docker/
```

---

## Notes

- WireGuard private keys are inside the sysupgrade tar and UCI export — treat these files as sensitive
- `.env` files backed up to NAS only — never push to GitHub
- Run `snapraid sync` after backup completes to protect new files on NAS
- SSH to NAS as root requires temporary `PermitRootLogin yes` — re-disable after rsync

---

## Known Limitations

- No automated scheduling — manual procedure only
- No offsite backup — NAS is single location
- LXC volumes (Grafana data, Loki data) not included — config only

**Planned:** automate with cron + offsite copy via rclone (future)
