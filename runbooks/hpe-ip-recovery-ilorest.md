# Runbook: HPE Intelligent Provisioning Recovery via ilorest (Gen10)

**Applies to:** HPE ProLiant Gen10 servers with empty or corrupt NAND (IP not registered)  
**Prerequisites:** iLO 5 network access, iLO Administrator credentials, Linux workstation on same LAN  
**Time:** ~20 minutes

---

## When to Use This

Use this runbook when:
- F10 at POST does nothing or is absent
- IP is not listed as a menu item in the iLO web interface
- Recovery ISO boots but exits immediately before reaching the graphical stage
- iLO firmware inventory shows an IP version but the partition is not functional

---

## Step 1 — Install ilorest

On your Linux management workstation (Ubuntu/Debian):

```bash
# Register HPE's public GPG key securely
curl -fsSL https://downloads.linux.hpe.com/SDR/hpePublicKey2048_key1.pub | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hpe-ilorest.gpg > /dev/null

# Add the official ilorest repository
echo "deb [signed-by=/usr/share/keyrings/hpe-ilorest.gpg] \
  https://downloads.linux.hpe.com/SDR/repo/ilorest jammy/current non-free" | \
  sudo tee /etc/apt/sources.list.d/ilorest.list

# Update package list and install
sudo apt update
sudo apt install ilorest
```

---

## Step 2 — Authenticate to iLO

```bash
ilorest login <ILO_IP> -u <ILO_USER> -p <ILO_PASSWORD>
```

---

## Step 3 — Clear the Corrupted NAND State

This command purges the iLO RESTful API state, forcing a logical format of the NAND flash blocks and rebuilding the IP partition registry in the background:

```bash
ilorest clearrestapistate
```

> **Engineering note:** This approach bypasses a limitation in the iLO 5 web interface, which hides the physical "Format" button when the hardware self-test (`ASIC Fuses` / `Embedded Flash`) reports healthy electrical status — even when the IP Linux filesystem partition table is logically corrupt. `clearrestapistate` reaches the underlying state directly via the RESTful API regardless of what the web UI exposes.

**Expected behaviour:** The iLO processor reboots automatically after this command, dropping the active session. Wait 2–3 minutes until the board responds to ping before proceeding.

---

## Step 4 — Cold Boot the Server

Re-authenticate after iLO comes back online, then force a cold boot to flush residual NVRAM pointers in the system ROM:

```bash
# Re-authenticate after iLO restarts
ilorest login <ILO_IP> -u <ILO_USER> -p <ILO_PASSWORD>

# Force a cold boot — clears system ROM inventory pointers
ilorest reboot ColdBoot

# Clean up local session
ilorest logout
```

---

## Step 5 — Boot the IP Recovery ISO

### Option A — USB (recommended for reliability)

```bash
# On Linux workstation — dd the ISO to USB
sudo dd if=IP392.2026_0214.3.iso of=/dev/sdX bs=4M status=progress oflag=sync
```

Boot sequence on server:
1. POST → F11 → One-Time Boot Menu
2. Select "Run a UEFI application from a file system"
3. Select the USB device
4. Navigate to and execute `EFI\BOOT\BOOTX64.EFI`

### Option B — iLO Virtual Media URL

Requires a stable HTTP server on the same LAN as iLO. Use nginx, not Python's http.server (range request incompatibility):

```bash
# On workstation — serve ISO directory
docker run -d --rm -v $(pwd):/usr/share/nginx/html:ro -p 8080:80 nginx

# SSH into iLO
ssh Administrator@<iLO_IP>

# Mount and set boot (order matters)
vm cdrom insert http://<workstation_IP>:8080/<iso_filename>.iso
vm cdrom set connect
vm cdrom set boot_once
power reset

# Verify state before reset
vm cdrom get
# Expected: Image Connected = Yes, Boot Option = BOOT_ONCE
```

---

## Step 6 — Complete the IP Installation

Once the installer loads:

1. Click inside the iLO HTML5 console canvas to capture keyboard focus
2. Navigate the EULA screen: `Tab` → `Tab` → `Space` (check box) → `Tab` → `Enter`
3. Follow the First Time Setup wizard
4. Reboot when prompted — F10 should now be active at POST

---

## Step 7 — Verify

In the iLO web interface:
- Lifecycle Management menu item should be visible
- Firmware Information → Intelligent Provisioning should show v3.92.5 (or current)

At POST:
- F10 prompt should appear during boot

---

## Troubleshooting

| Symptom                                                      | Likely Cause                                                       | Fix                              |
|--------------------------------------------------------------|--------------------------------------------------------------------|----------------------------------|
| Installer enters and exits immediately                       | NAND CHIF handshake failing — NVRAM not cleared                    | Repeat Step 3                    |
| `vm cdrom set boot_once` returns COMMAND PROCESSING FAILED   | `connect` not run before `boot_once`                               | Run `vm cdrom set connect` first |
| iLO shows IP version in firmware inventory but F10 is absent | Cached manifest entry — partition is empty                         | This runbook applies             |
| Tofu squares on installer screen                             | Font cache init failure (display resolution mismatch at ROM v2.68) | Click canvas for keyboard focus; navigate blind with Tab/Space/Enter |
| Java console crashes                                         | OpenJDK 11+ incompatible with HPE legacy Java applet               | Use HTML5 console only           |

---

## References

- HPE iLO RESTful Interface Tool (ilorest): https://github.com/HewlettPackard/python-redfish-utility
- HPE IP Recovery Media: https://support.hpe.com (search ML110 Gen10 → Intelligent Provisioning)
- Incident log: `incidents/2026-06-10_ml110-gen10-ip-recovery.md`
