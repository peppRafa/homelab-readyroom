# Hardware Profile: HPE ProLiant ML110 Gen10 — Units M1 & M2

**Category:** Tower Server  
**Acquisition:** Used enterprise resale lot  
**Role:** Homelab compute nodes (temporary) → resale

---

## Unit Overview

|      Field    | Machine 1 (M1)       |  Machine 2 (M2)        |
|---------------|----------------------|------------------------|
| iLO Hostname  | Sovereing            | Nebula                 |
| iLO IPv4      | Deep Space 9         | Earth Station Mckinley |
| System ROM    | U33 v2.68 (Jul 2022) | U33 v2.68 (Jul 2022)   |
| iLO Firmware  | 5 v3.19 (Apr 2026)   | 5 v3.19 (Apr 2026)     |
| IP Version    | 3.92.5               | 3.92.5                 |
| SPS Firmware  | 4.1.4.804            | 4.1.4.804              |

*Partially redacted for public repo.*

---

## Firmware Inventory (both units — identical)

| Component                    | Version                | Notes                        |
|------------------------------|------------------------|------------------------------|
| System ROM                   | U33 v2.68 (07/14/2022) | Update pending: v3.66        |
| Redundant System ROM         | U33 v2.52 (07/08/2021) | Recovery set                 |
| iLO 5                        | 3.19 (Apr 07 2026)     | Current stable               |
| SPS Firmware                 | 4.1.4.804              | Update pending: 04.01.05.201 |
| Smart Array P408i-a SR Gen10 | 5.61                   | OK                           |
| HPE Ethernet 1Gb 2-port 332i | 20.14.54               | OK                           |
| Power Management Controller  | 1.0.8                  | OK                           |
| SPLD                         | 0x17                   | OK                           |
| Embedded Video Controller    | 2.5                    | OK                           |
| TPM                          | 73.20                  | OK                           |
| Innovation Engine (IE)       | 0.2.3.0                | OK                           |
| Intelligent Provisioning     | 3.92.5                 | Registered — F10 active      |

---

## Storage — Machine 1

| Bay   | Device                   | Size   | Status              | Notes                             |
|-------|--------------------------|--------|---------------------|-----------------------------------|
| Bay 1 | HPE SAS 10k 12gbs sff    | 1,2tb  | Predictive Failure  | Pending ssacli verification       |
| Bay 2 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal              | Flag present — SMART test pending |
| Bay 3 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal              | Pending ssacli verification       |
| Bay 4 | HPE SAS 10k 12gbs sff    | 1,2tb  | Predictive Failure  | Flag present — SMART test pending |
| Bay 5 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal              | Pending ssacli verification       |

## Storage — Machine 2

| Bay   | Device                   | Size   | Status | Notes                       |
|-------|--------------------------|--------|--------|-----------------------------|
| Bay 1 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal | Pending ssacli verification |
| Bay 2 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal | Pending ssacli verification |
| Bay 3 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal | Pending ssacli verification |
| Bay 4 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal | Pending ssacli verification |
| Bay 5 | HPE SAS 10k 12gbs sff    | 1,2tb  | Normal | Pending ssacli verification |

> SMART testing to be performed via SystemRescue + ssacli after Ubuntu Server install or via live USB.
> Controller: HPE Smart Array P408i-a SR Gen10 (embedded, Slot 0)  
> RAID: Not configured. All drives presented as unassigned physical devices.

---

## Known Issues & Flags

### Both Units
- **System ROM update required:** v2.68 → v3.66. Must be paired with SPS 04.01.05.201. Use SPP bootable ISO for safe ordered update.
- **Embedded Diagnostics 0x0D crash:** UEFI Embedded Diagnostics returns X64 General Protection Exception at current ROM version. Confirmed firmware bug, not hardware. Resolves post-ROM update.

### Machine 1
- Predictive failure flags on Bay 2 and Bay 4. Drives not yet SMART-tested. Do not use Bay 2/4 drives in any array until test results confirmed.

---

## Access

| Method           | Address                          | Notes                                   |
|------------------|----------------------------------|-----------------------------------------|
| iLO Web UI (M1)  | https://192.168.10.101           | iLO Advanced license active             |
| iLO Web UI (M2)  | https://192.168.10.100           | iLO Advanced license active             |
| iLO SSH (M2)     | ssh Administrator@192.168.10.100 | Validated                               |
| WireGuard remote | Via 10.0.0.x tunnel              | KVM console accessible remotely         |
| F10 (IP)         | POST screen                      | Registered and functional on both units |

---

## Pending Tasks Checklist

- [ ] Install Ubuntu Server 24.04 LTS (both units)
- [ ] Run SPP via iLO virtual media — update System ROM v3.66 + SPS 04.01.05.201
- [ ] Re-run Embedded Diagnostics post-ROM update
- [ ] Run `ssacli ctrl slot=0 pd all show detail` — populate storage table above
- [ ] Run SMART test on Bay 2 and Bay 4 (M1) — determine drive viability
- [ ] Run Memtest86+ (2 passes minimum, both units) — save HTML report
- [ ] Download AHS log from iLO (both units) — archive for resale documentation
- [ ] Generate ssacli ADU report (both units) — archive for resale documentation
