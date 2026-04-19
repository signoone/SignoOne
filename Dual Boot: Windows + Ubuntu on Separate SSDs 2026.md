# Dual Boot: Windows + Ubuntu on Separate SSDs Guide 2026
> Two separate SSDs. Windows on Drive 1. Ubuntu on Drive 2. Independent bootloaders. Clean setup.
<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/8cb25fa9-2bf2-4858-bfa7-4a37e2eb265d" />

---

## Table of Contents
1. [How It Works](#how-it-works)
2. [Pre-Installation Checklist](#pre-installation-checklist)
3. [Create Ubuntu USB](#create-ubuntu-usb)
4. [Boot the USB](#boot-the-usb)
5. [Ubuntu Installation — Manual Partitioning](#ubuntu-installation--manual-partitioning)
6. [Post-Installation](#post-installation)
7. [Understanding UEFI Boot Selection](#understanding-uefi-boot-selection)
8. [Troubleshooting](#troubleshooting)

---

## How It Works

### The Goal
Each SSD is fully independent with its own bootloader:

```
Drive 1 (Windows SSD)
├── ESP (EFI System Partition)
│   └── Windows Boot Manager  ← boots Windows
├── Microsoft Reserved
├── Windows C: partition
└── Windows Recovery

Drive 2 (Ubuntu SSD)
├── ESP (EFI System Partition)
│   └── GRUB                  ← boots Ubuntu (and detects Windows)
├── swap
├── / (root)
└── /home
```

Your UEFI firmware sits before everything. On power-on, UEFI checks its boot priority list and hands control to whichever bootloader is set first.

### Why Separate ESPs (Not Shared)
By default, Ubuntu installs GRUB onto the **Windows SSD's existing ESP**. That works, but:
- Your Ubuntu drive now depends on the Windows drive to boot
- Removing either drive can break boot
- GRUB files are sitting inside a Windows partition

With separate ESPs, each drive is completely self-contained. Unplug either drive — the other still boots perfectly. This is the clean setup.

---

## Pre-Installation Checklist

### 1. Disable Fast Startup in Windows
**Do this first before anything else.**

```
Start Menu → Settings → System → Power & Sleep
→ Additional Power Settings
→ Choose what the power buttons do
→ Turn on fast startup → UNCHECK IT
→ Save changes
```

> **Why:** Windows Fast Startup hibernates the disk instead of fully shutting down. If Ubuntu touches your Windows partition while it's hibernated, it can corrupt your Windows filesystem — permanently.

---

### 2. Identify Your Drives in Windows
Open Disk Management before installing Ubuntu:
```
Right-click Start → Disk Management
```

Note which disk is which:
- **Disk 0** — Windows SSD (has a small EFI System Partition ~100MB)
- **Disk 1** — Ubuntu SSD (empty or will be wiped)

During Ubuntu installation, drives appear as `/dev/sda`, `/dev/sdb`, etc. You need to know which physical drive maps to which. Getting this wrong means potentially formatting your Windows drive.

---

### 3. Disable Fast Boot in UEFI
Separate from Windows Fast Startup. This is in your motherboard firmware.

- Restart → spam **F2 / DEL / F12** (varies by motherboard brand) to enter UEFI
- Find **Fast Boot** → Disable
- Save and exit

> **Why:** Fast Boot skips USB detection. Your Ubuntu USB may not appear as a boot option.

---

### 4. Disable Secure Boot in UEFI
While in UEFI settings:
- Find **Secure Boot** (usually under Security or Boot tab)
- Set to **Disabled**
- Save and exit

> **Why:** Secure Boot can block GRUB from loading or throw signature verification errors during install. Not worth the complexity for a personal dual boot setup.

---

### 5. Wipe / Clear the Ubuntu SSD
If the second SSD has existing data, back it up now. The installation will wipe it.

---

## Create Ubuntu USB

1. Download **Ubuntu 26.04 LTS** ISO from [ubuntu.com](https://ubuntu.com)
2. Download **Rufus** (Windows) from [rufus.ie](https://rufus.ie)
3. Flash settings in Rufus:
   - Partition scheme: **GPT**
   - Target system: **UEFI (non CSM)**
   - File system: FAT32

> **Critical:** Do not use MBR or Legacy/CSM mode. This must match your UEFI setup or the bootloader partition scheme will be wrong.

---

## Boot the USB

Restart your PC. Spam your **one-time boot menu key** (not BIOS setup):

| Brand | Key |
|-------|-----|
| ASUS | F8 or F12 |
| Gigabyte | F12 |
| MSI | F11 |
| ASRock | F11 |
| Dell | F12 |
| HP | F9 |

In the boot menu, select your USB. **Verify it says UEFI before the name:**

```
✅  UEFI: SanDisk USB          ← select this
❌  SanDisk USB                ← legacy/CSM mode, do NOT use
```

> If you accidentally boot in legacy mode, your partition setup will be wrong. Stop, reboot, and select the correct UEFI entry.

---

## Ubuntu Installation — Manual Partitioning

Go through the first few installer screens normally (language, keyboard, Wi-Fi) until you reach **Installation Type**.

---

### Step 1: Choose "Something Else"

On the Installation Type screen, select:
- **Something else** (also labeled "Manual partitioning" or "Custom install" depending on Ubuntu version)

Do **not** select:
- "Install Ubuntu alongside Windows" — this uses the default shared ESP behavior
- "Erase disk and install Ubuntu" — gives you no control over which disk

---

### Step 2: Identify Your Drives in the Partition Table

You will see a list of all drives and partitions. It looks like this:

```
/dev/sda          ← Windows SSD — DO NOT TOUCH THIS
  /dev/sda1       EFI System Partition    100MB    fat32
  /dev/sda2       Microsoft Reserved      16MB
  /dev/sda3       [Windows C: drive]      [large]  ntfs
  /dev/sda4       Windows Recovery        ~500MB

/dev/sdb          ← Ubuntu SSD — work here only
  free space      [entire drive]
```

**Verify your drive assignments before clicking anything.** The sizes listed will help you confirm which is which.

---

### Step 3: Create Partitions on the Ubuntu SSD (/dev/sdb)

Click on the free space under `/dev/sdb` and create the following partitions in order:

---

#### Partition 1 — EFI System Partition (ESP)
This is the critical partition. It gives the Ubuntu SSD its own independent bootloader.

| Setting | Value |
|---------|-------|
| Size | 512 MB |
| Type | Primary |
| Format | FAT32 |
| Mount point | `/boot/efi` |
| Flag | EFI System Partition ✓ |

> **Do not skip this.** Without it, GRUB will default to the Windows SSD's ESP and the two drives will not be independent.

---

#### Partition 2 — Swap

| Setting | Value |
|---------|-------|
| Size | Equal to your RAM (for hibernation support) OR half your RAM (no hibernation needed) |
| Type | Primary |
| Format | `swap area` (select from dropdown, not ext4) |
| Mount point | none |

*Example: 16GB RAM → 16GB swap if you want hibernation, 8GB if you don't.*

---

#### Partition 3 — Root ( / )
This is where Ubuntu itself is installed.

| Setting | Value |
|---------|-------|
| Size | 50GB minimum. 80–100GB recommended. More if you are a developer. |
| Type | Primary |
| Format | ext4 |
| Mount point | `/` |

---

#### Partition 4 — Home ( /home )
Separates personal files from the OS. Optional but strongly recommended.

| Setting | Value |
|---------|-------|
| Size | All remaining space |
| Type | Primary |
| Format | ext4 |
| Mount point | `/home` |

> **Why /home matters:** If you ever reinstall Ubuntu, you can reformat only the root partition and keep your `/home` partition untouched. Your personal files survive the reinstall.

---

#### Final Partition Layout on /dev/sdb

```
/dev/sdb
├── /dev/sdb1    FAT32    512MB       /boot/efi   ← ESP (bootloader lives here)
├── /dev/sdb2    swap     [RAM size]  [swap]
├── /dev/sdb3    ext4     80GB        /
└── /dev/sdb4    ext4     remaining   /home
```

---

### Step 4: Set the Boot Loader Location

At the bottom of the partitioning screen is a dropdown:

```
Device for boot loader installation: [          ▼]
```

**This defaults to /dev/sda (Windows drive). Change it.**

```
❌  /dev/sda    ← Windows drive — do not touch
✅  /dev/sdb    ← Ubuntu drive — select this
```

Select the **whole Ubuntu drive** (`/dev/sdb`), not a specific partition like `/dev/sdb1`.

> **This is the most commonly missed step.** If left as `/dev/sda`, GRUB installs into the Windows ESP and the drives are no longer independent.

---

### Step 5: Review and Install

Before clicking Install Now, verify:

- [ ] `/dev/sda` — untouched, no changes shown
- [ ] `/dev/sdb1` — 512MB FAT32, mounted at `/boot/efi`, flagged as EFI System Partition
- [ ] `/dev/sdb2` — swap
- [ ] `/dev/sdb3` — ext4, mounted at `/`
- [ ] `/dev/sdb4` — ext4, mounted at `/home`
- [ ] Boot loader device — set to `/dev/sdb`

If all correct, click **Install Now**.

---

## Post-Installation

### First Boot

If the system boots straight into Windows after installation:
1. Restart and enter UEFI settings (F2/DEL)
2. Find **Boot Order** or **Boot Priority**
3. Move your Ubuntu SSD above the Windows SSD
4. Save and exit

GRUB will load and show you a boot menu.

---

### Enable os-prober (Make GRUB Detect Windows)

Ubuntu 22.04+ disables os-prober by default. Without enabling it, GRUB will not show Windows in its boot menu.

Open a terminal in Ubuntu and run:

```bash
sudo nano /etc/default/grub
```

Find or add this line:
```bash
GRUB_DISABLE_OS_PROBER=false
```

Save the file (`Ctrl+O` → Enter → `Ctrl+X`), then run:

```bash
sudo update-grub
```

Expected output includes:
```
Found Windows Boot Manager on /dev/sda1@/EFI/Microsoft/Boot/bootmgfw.efi
```

That confirms GRUB has detected Windows. On next reboot, the GRUB menu will show both Ubuntu and Windows.

---

### GRUB Boot Menu Behaviour

```
┌─────────────────────────────┐
│  Ubuntu 26.04 LTS           │  ← default, auto-boots after timeout
│  Advanced options for Ubuntu│
│  Windows Boot Manager       │  ← select this to boot Windows
└─────────────────────────────┘
```

Default timeout is 10 seconds. To change it:

```bash
sudo nano /etc/default/grub
# Change GRUB_TIMEOUT=10 to whatever you want (seconds)
sudo update-grub
```

---

## Understanding UEFI Boot Selection

### What UEFI Does
UEFI runs before any operating system. On power-on, it reads its boot priority list and hands control to the first working bootloader it finds.

### Your Boot Priority List in UEFI

```
Boot Priority:
1. Samsung SSD 2 (GRUB)              ← Ubuntu drive, set as default
2. Samsung SSD 1 (Windows Boot Mgr)  ← Windows drive, fallback
3. USB Drive
```

### Two Ways to Switch Between OSes

**Method A — GRUB Menu (Recommended)**
- Ubuntu SSD is set as boot priority #1
- GRUB loads every time and shows a menu with Ubuntu and Windows
- Select with arrow keys. Auto-boots Ubuntu if you do nothing.

**Method B — One-time UEFI Boot Menu**
- Windows SSD is set as priority #1
- Every time you want Ubuntu, hit F12 at startup for a one-time boot selection
- Select the Ubuntu SSD from the list
- More manual, but useful if you primarily use Windows

### Independence of Each Drive

With this setup, unplugging either drive has no effect on the other:

```
Remove Ubuntu SSD → Windows boots normally via Windows Boot Manager
Remove Windows SSD → Ubuntu boots normally via GRUB
```

This is the key advantage of separate ESPs over a shared one.

---

## Troubleshooting

### GRUB not showing after install
- Enter UEFI and check boot order — Ubuntu SSD may not be first
- Verify the ESP was created on the Ubuntu SSD (`/dev/sdb1`)
- Confirm boot loader was set to `/dev/sdb` during installation, not `/dev/sda`

### Windows not showing in GRUB menu
- Run `sudo update-grub` in Ubuntu terminal
- If Windows still doesn't appear, verify `GRUB_DISABLE_OS_PROBER=false` is in `/etc/default/grub`
- Run `sudo apt install os-prober` if os-prober is not installed, then run `sudo update-grub` again

### Booted USB in legacy mode accidentally
- The partition scheme will be wrong (MBR instead of GPT)
- Restart the installation from scratch
- Make sure to select the `UEFI:` prefixed USB entry in the boot menu

### GRUB rescue prompt appears
- This usually means GRUB cannot find its config file
- Boot from Ubuntu USB, select "Try Ubuntu"
- Open terminal and run:
```bash
sudo mount /dev/sdb3 /mnt
sudo mount /dev/sdb1 /mnt/boot/efi
sudo grub-install --target=x86_64-efi --efi-directory=/mnt/boot/efi --boot-directory=/mnt/boot /dev/sdb
sudo update-grub
```

### Windows partition inaccessible from Ubuntu
- Fast Startup was not disabled in Windows
- Boot into Windows, disable Fast Startup (see Pre-Installation section), and do a full shutdown (not restart)

---

## Quick Reference

### Key Commands in Ubuntu

```bash
# Regenerate GRUB config (run after any GRUB change)
sudo update-grub

# Edit GRUB settings
sudo nano /etc/default/grub

# Check which drive is which
lsblk

# View partition details
sudo fdisk -l

# Check mounted partitions
df -h
```

### Drive Naming Convention

| Name | Meaning |
|------|---------|
| `/dev/sda` | First detected SSD/HDD |
| `/dev/sdb` | Second detected SSD/HDD |
| `/dev/sda1` | First partition on first drive |
| `/dev/sdb1` | First partition on second drive |

> Drive letter assignment (`sda` vs `sdb`) can vary depending on which port the drives are plugged into and UEFI detection order. Always verify by checking sizes in the partition table before making changes.

---

*Ubuntu 26.04 LTS (Resolute Raccoon) — Released April 23, 2026*
