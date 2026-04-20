# VMware Workstation 25H2u1 — Ubuntu Installation Guide
**Documented from live troubleshooting session**  
File: `VMware-Workstation-Full-25H2u1-25219725.x86_64.bundle`
`Download`: https://support.broadcom.com/group/ecx/free-downloads
OS: Ubuntu (VMware Workstation host)

---

## Table of Contents
1. [Prerequisites](#1-prerequisites)
2. [Installation](#2-installation)
3. [Error: vmmon module not loaded](#3-error-vmmon-module-not-loaded)
4. [Diagnosis: vmware-modconfig fails](#4-diagnosis-vmware-modconfig-fails)
5. [Root Cause: Secure Boot blocking modules](#5-root-cause-secure-boot-blocking-modules)
6. [Fix Option A: Disable Secure Boot (Recommended)](#6-fix-option-a-disable-secure-boot-recommended)
7. [Fix Option B: Sign Modules (Stay on Secure Boot)](#7-fix-option-b-sign-modules-stay-on-secure-boot)
8. [Verification](#8-verification)

---

## 1. Prerequisites

Before running the installer, ensure build tools and kernel headers are installed:

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) gcc make
```

---

## 2. Installation

### Make the bundle executable

```bash
chmod +x VMware-Workstation-Full-25H2u1-25219725.x86_64.bundle
```

### Run the installer

**With GUI (desktop environment):**
```bash
sudo ./VMware-Workstation-Full-25H2u1-25219725.x86_64.bundle
```

**Without GUI (headless/CLI only):**
```bash
sudo ./VMware-Workstation-Full-25H2u1-25219725.x86_64.bundle --console
```

---

## 3. Error: vmmon module not loaded

**Symptom:** VMware launches but immediately throws a popup:

```
Error
Could not open /dev/vmmon: No such file or directory.
Please make sure that the kernel module 'vmmon' is loaded.
```

> 📸 Screenshot: `error-vmmon-not-loaded.png`
> <img width="609" height="529" alt="Screenshot From 2026-04-18 16-51-06" src="https://github.com/user-attachments/assets/91ac9fa3-aa1f-4278-8559-3dc5be1b87e6" />


**What this means:** VMware installed correctly, but its kernel modules (`vmmon`, `vmnet`) are either not compiled or not loaded.

---

## 4. Diagnosis: vmware-modconfig fails

Run the module recompile and reinstall command:

```bash
sudo vmware-modconfig --console --install-all
```

**Output observed:**

```
Stopping VMware services:
    VMware Authentication Daemon                done
    Virtual machine monitor                     done
Starting VMware services:
    Virtual machine monitor                     failed
    Virtual machine communication interface     done
    VM communication interface socket family    done
    Virtual ethernet                            failed
    VMware Authentication Daemon                done
Unable to start services
```


**What this means:** `vmmon` (Virtual machine monitor) and `vmnet` (Virtual ethernet) modules compiled but failed to load at runtime.

### Confirm the exact failure

```bash
sudo modprobe vmmon
sudo modprobe vmnet
```

**Output observed:**

```
modprobe: ERROR: could not insert 'vmmon': Key was rejected by service
modprobe: ERROR: could not insert 'vmnet': Key was rejected by service
```

### Check Secure Boot status

```bash
mokutil --sb-state
```

**Output observed:**

```
SecureBoot enabled
```


**Confirmed root cause:** Secure Boot is active. It rejects unsigned kernel modules. VMware's `vmmon` and `vmnet` modules are unsigned, so the kernel refuses to load them.

---

## 5. Root Cause: Secure Boot blocking modules

Secure Boot enforces a chain of trust — only kernel modules signed with a trusted key are allowed to load. VMware ships unsigned modules, so as long as Secure Boot is enabled, `vmmon` and `vmnet` will always be rejected with `Key was rejected by service`.

This is the **most common reason** VMware fails on modern Ubuntu installs.

---

## 6. Fix Option A: Disable Secure Boot (Recommended)

**Best for:** Most users. Especially on machines used as VMware hosts where Secure Boot provides minimal real-world security benefit.

### Steps

1. Reboot the machine
2. Enter UEFI/BIOS firmware (spam `F2`, `Del`, `Esc`, or `F10` during POST — varies by motherboard)
3. Navigate to **Security** or **Boot** tab
4. Find **Secure Boot** → set to **Disabled**
5. Save and exit
6. Boot back into Ubuntu
7. Load the modules and start VMware:

```bash
sudo modprobe vmmon
sudo modprobe vmnet
vmware
```

---

## 7. Fix Option B: Sign Modules (Stay on Secure Boot)

**Best for:** Users who need Secure Boot enabled for compliance or security reasons.

> ⚠️ Note: You will need to redo module signing after every VMware update that ships new modules.

### Steps

```bash
# Step 1: Create a Machine Owner Key (MOK)
openssl req -new -x509 -newkey rsa:2048 \
  -keyout ~/MOK.priv \
  -out ~/MOK.der \
  -days 36500 \
  -subj "/CN=VMware-Modules/" \
  -nodes

# Step 2: Sign vmmon and vmnet with your key
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 \
  ~/MOK.priv ~/MOK.der $(modinfo -n vmmon)

sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 \
  ~/MOK.priv ~/MOK.der $(modinfo -n vmnet)

# Step 3: Enroll the key into Secure Boot
sudo mokutil --import ~/MOK.der
```

You will be prompted to set a password. **Remember it** — you'll need it on the next boot.

```bash
sudo reboot
```

### On reboot — MOK enrollment screen

A blue **MOK Management** screen will appear (do NOT let it time out):

1. Select **Enroll MOK**
2. Select **Continue**
3. Select **Yes**
4. Enter the password you set in Step 3
5. Select **Reboot**

### After rebooting back into Ubuntu

```bash
sudo modprobe vmmon
sudo modprobe vmnet
vmware
```

---

## 8. Verification

After applying either fix, verify modules are loaded:

```bash
lsmod | grep vmmon
lsmod | grep vmnet
```

Both should return output. If they return nothing, the modules are not loaded.

Launch VMware:

```bash
vmware
```

If VMware opens without the `/dev/vmmon` error — you're done.

---

## Quick Reference: Common Failure Points

| Symptom | Cause | Fix |
|---|---|---|
| `Could not open /dev/vmmon` | Module not loaded | Load with `modprobe` or fix underlying cause |
| `Virtual machine monitor: failed` in modconfig | Module won't load | Check Secure Boot or kernel header mismatch |
| `Key was rejected by service` | Secure Boot enabled | Disable Secure Boot OR sign the modules |
| `kernel headers not found` | Missing build deps | `apt install linux-headers-$(uname -r)` |
| `gcc version mismatch` | Wrong gcc version | Match gcc version to kernel build gcc |

---

## Commands Cheat Sheet

```bash
# Check kernel version
uname -r

# Install build dependencies
sudo apt install -y build-essential linux-headers-$(uname -r) gcc make

# Recompile and reinstall VMware modules
sudo vmware-modconfig --console --install-all

# Load modules manually
sudo modprobe vmmon
sudo modprobe vmnet

# Check if modules are loaded
lsmod | grep vmmon
lsmod | grep vmnet

# Check Secure Boot status
mokutil --sb-state

# Launch VMware
vmware
```

---

*Documentation generated from live troubleshooting. Screenshots referenced inline were captured during the session.*
