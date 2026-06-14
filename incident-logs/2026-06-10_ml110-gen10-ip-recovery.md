# Incident Log: HPE ProLiant ML110 Gen10 — Intelligent Provisioning Recovery

**Date Range:** 2026-06-04 to 2026-06-14  
**Severity:** Medium (lab hardware, no production impact)  
**Status:** Resolved  
**Affected Units:** Machine 1 (Sovering) · Machine 2 (Nebula)

---

## Background

Two HPE ProLiant ML110 Gen10 servers acquired for homelab use and resale evaluation. Both units arrived with Intelligent Provisioning (IP) in a non-functional state. IP is HPE's embedded provisioning and maintenance framework, accessible via F10 at POST, required for OS deployment and firmware maintenance workflows.

---

## Root Cause

Both units had System ROM at version U33 v2.68 (July 2022). The pre-installed IP version (v3.31) was structurally incompatible with this BIOS revision and produced a kernel panic on launch. The embedded NAND flash was subsequently formatted via the iLO 5 diagnostics menu to clear a latent NVRAM write-lock — successfully resolving the locked sectors but wiping all IP software files and UEFI boot registration entries.

---

## Recovery Attempts (Machine 2 — documented in detail)

### Attempt 1 — Manual FAT32 File Copy
Extracted IP recovery ISO contents to an MBR FAT32 USB drive, targeting the BIOS "Select a device to update" maintenance menu.  
**Result: Failed.** The motherboard maintenance layer expects standalone binary components. The IP ISO is an encrypted self-contained Linux kernel wrapper with no loose `.flash` or `.bin` files on its root. No `.fwpkg` component exists as a separate download on HPE support — the recovery ISO is the only delivery vehicle.

### Attempt 2 — Raw Sector Clone + UEFI Shell
Block-level clone via `dd` to USB. Attempted boot via F11 and Embedded UEFI Shell, navigating to `FS0:`.  
**Result: Failed.** The HPE ISO uses a hybrid multi-partition filesystem that the UEFI file manager could not index. `ls` returned `File Not Found` in the shell.

### Attempt 3 — Manual UEFI Boot Path Override
One-Time Boot Menu (F11) → "Run a UEFI application from a file system" → selected USB → manually executed `EFI\BOOT\BOOTX64.EFI`.  
**Result: Partial success.** Bypassed the boot loop and loaded the IP installer environment into RAM. Installer reached the graphical stage but all text rendered as hollow square blocks (tofu effect).

**Tofu rendering analysis:** The iLO HTML5 console is a remote framebuffer — it streams pixel data generated server-side. AdGuard Home and browser font loading are irrelevant. The tofu is produced by the IP Linux microkernel's font cache failing to initialize, most likely due to a display resolution mismatch at BIOS U33 v2.68. A physical VGA monitor would show identical output.

**Keyboard input issue:** Tab key was captured by the browser's own focus management rather than passed to the canvas. Fix: click inside the canvas first to give it explicit mouse focus, then navigate. Full-screen mode (iLO console expand icon) suppresses browser Tab interception.

### Attempt 4 — Java Web Start Console
Attempted native Java client via `icedtea-netx` on Linux Mint to bypass the HTML5 canvas keyboard issue.  
**Result: Failed.** Crashed immediately with `java.lang.NoSuchMethodError` — modern OpenJDK 11+ removed internal legacy X.509 encryption libraries that HPE's legacy Java applet depends on.

### Attempt 5 — iLO Virtual Media URL Mount
Served the IP recovery ISO over HTTP from the i5 host (192.168.10.140) and mounted it via iLO SSH CLI as a virtual CDROM, bypassing USB entirely.

```bash
# Correct command sequence (order matters)
vm cdrom insert http://192.168.10.140:8080/IP392.2026_0214.3.iso
vm cdrom set connect
vm cdrom set boot_once
power reset
```

**Common mistake:** Running `vm cdrom set boot_once` before `vm cdrom set connect` returns `COMMAND PROCESSING FAILED`. The insert command creates the object; connect activates it; boot_once then works.

**HTTP server requirement:** Python's built-in `http.server` caused iLO connection failures during boot-time ISO streaming (event log: "Could not establish a connection"). Replace with nginx for reliable range-request handling:

```bash
docker run -d --rm -v $(pwd):/usr/share/nginx/html:ro -p 8080:80 nginx
```

**Result:** ISO mounted, boot menu showed virtual CD, installer entered and exited immediately before reaching graphical stage.

**Analysis of rapid exit:** IP installer loads, attempts CHIF handshake with iLO to register itself to NAND, fails when NAND partition is empty/unformatted, and aborts. This is distinct from the tofu screen issue — the installer never reached the graphical stage in this state.

**Note on Firmware Information display:** The iLO firmware inventory showed IP 3.92.5 as "installed" throughout this process. This is a cached manifest entry, not confirmation of a live partition. The NAND was empty; iLO was reporting the last known version from its registry.

### Attempt 6 — iLO RESTful API / ilorest (Resolution)
Used the HPE iLO RESTful Interface Tool (`ilorest`) to format the NVRAM partition directly, clearing the stale registry state and allowing the recovery installer to complete the CHIF registration handshake on the next boot.

**Result: Success.** IP 3.92.5 installed and fully registered on both machines. F10 prompt active. iLO web interface shows IP menu item under Lifecycle Management.

---

## Post-Recovery State

| Component                 | Version              | Status                        |
|---------------------------|----------------------|-------------------------------|
| iLO 5 Firmware            | 3.19 (Apr 2026)      | OK                            |
| System ROM                | U33 v2.68 (Jul 2022) | Update available: v3.66       |
| SPS Firmware              | 4.1.4.804            | Update required: 04.01.05.201 |
| Intelligent Provisioning  | 3.92.5               | OK — registered               |
| Smart Array P408i-a       | 5.61                 | OK                            |
| Embedded Video Controller | 2.5                  | OK                            |

---

## Pending Actions

- **Firmware update (both units):** System ROM v3.66 requires paired SPS 04.01.05.201. Manual SPS flashing on Gen10 has caused non-bootable systems in the field when done out of order. Recommended path: Service Pack for ProLiant (SPP) bootable ISO via iLO virtual media — handles update ordering automatically.
- **Embedded Diagnostics 0x0D error:** During diagnostic testing, the UEFI Embedded Diagnostics produced an X64 Exception Type 0x0D (General Protection Exception). Confirmed as a firmware bug at ROM v2.68, not a hardware fault. Resolves after System ROM update to v3.66.
- **Drive SMART verification:** Predictive failure flags present on Bay 2 and Bay 4 (Machine 1). Plan: boot SystemRescue, install `ssacli`, run `ctrl slot=0 pd all show detail` and `show smartstatus` per drive. Generate ADU report for documentation.
- **Memory test:** Memtest86+ bootable ISO, minimum 2 passes. Generates HTML report suitable for resale documentation.
- **OS installation:** Ubuntu Server 24.04 LTS (CLI-only). Enables online SPP mode, native `ssacli` and `ilorest` usage, and simplifies the full diagnostic workflow.

---

## Key Learnings

- The HPE IP recovery ISO contains no extractable `.fwpkg` component. It is the delivery vehicle itself, not a wrapper around a separate flashable file.
- `vm cdrom set connect` must precede `vm cdrom set boot_once` in the iLO SSH CLI sequence.
- Python `http.server` is unreliable for iLO virtual media URL streaming; nginx handles range requests correctly.
- iLO firmware inventory can display cached version numbers for components that are not actually installed on NAND.
- SPS firmware updates on Gen10 are high-risk if done manually without following HPE's prerequisite chain. Use SPP.
- The UEFI Embedded Diagnostics 0x0D error on Gen10 at ROM versions below v3.66 is a known firmware bug, not indicative of hardware failure.
