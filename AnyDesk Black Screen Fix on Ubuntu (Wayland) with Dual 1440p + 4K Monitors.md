# AnyDesk Black Screen Fix on Ubuntu (Wayland) with Dual 1440p + 4K Monitors
<img width="700" height="350" alt="image" src="https://github.com/user-attachments/assets/6ef73495-fe2a-4b32-a4f1-058f4b354f60" />


## Problem

AnyDesk connected to a remote machine, but one monitor displayed correctly while the other was completely black. The client machine (not the remote) was running Ubuntu with a dual monitor setup: one 1440p and one 4K display.

## Root Cause

Ubuntu was running a **Wayland** session. AnyDesk 8.0.2 on Wayland with mixed-DPI multi-monitor setups (1440p + 4K) fails to render on the high-DPI display. This is a known AnyDesk/Wayland incompatibility — not a hardware or connection issue.

**Confirmed via:**
```bash
echo $XDG_SESSION_TYPE
# Output: wayland
```

---

## System Info

| Property | Value |
|---|---|
| OS | Ubuntu 26.04 (Wayland-only) |
| AnyDesk Version | 8.0.2 |
| Client GPU | NVIDIA (xserver-xorg-video-nvidia-595) |
| Monitors | 1440p + 4K |
| Session Type | Wayland |

---

## What Was Tried and Failed

1. Disabling hardware acceleration in AnyDesk Settings → General (remote side) — no effect
2. Client-side AnyDesk Display settings changes — irrelevant, wrong side
3. Switching to X11 session via GDM3 — not viable, Ubuntu 26.04 is Wayland-only and `/usr/share/xsessions/` does not exist

---

## The Fix: Force AnyDesk to Use XWayland via GDK_BACKEND

Instead of switching the entire session to X11, force AnyDesk specifically to run through **XWayland** (the X11 compatibility layer built into Wayland).

### Step 1: Verify it works

Close AnyDesk, then run from terminal:

```bash
GDK_BACKEND=x11 anydesk
```

Connect to remote and confirm both monitors display correctly.

---

### Step 2: Make it permanent via the .desktop file

Edit the AnyDesk desktop entry:

```bash
sudo nano /usr/share/applications/anydesk.desktop
```

Find the `Exec` line:

```
Exec=/usr/bin/anydesk %u
```

Change it to:

```
Exec=env GDK_BACKEND=x11 /usr/bin/anydesk %u
```

> ⚠️ **Critical:** Do NOT write `Exec=Exec=...` — only one `Exec=` prefix. Double `Exec=` breaks the desktop entry and AnyDesk will disappear from the app launcher.

Save the file (`Ctrl+X` → `Y` → Enter).

---

### Step 3: Refresh the desktop database

```bash
sudo update-desktop-database
```

AnyDesk will now appear in the app launcher and always launch with the correct backend automatically.

---

## Final Verification

The correct `/usr/share/applications/anydesk.desktop` should look like this:

```ini
[Desktop Entry]
Type=Application
Name=AnyDesk
GenericName=AnyDesk
X-GNOME-FullName=AnyDesk
Exec=env GDK_BACKEND=x11 /usr/bin/anydesk %u
Icon=anydesk
Terminal=false
TryExec=anydesk
Categories=Network;GTK;
MimeType=x-scheme-handler/anydesk;
Name[de_DE]=AnyDesk
```

---

## Why This Works

`GDK_BACKEND=x11` tells AnyDesk's GTK layer to use X11 rendering instead of Wayland. Since Ubuntu still ships **XWayland** (an X11 compatibility server that runs inside the Wayland session), AnyDesk renders through XWayland cleanly — no full session switch required, no reboot, no system-wide changes. Everything else on your system stays on native Wayland.

---

## Notes

- This fix is per-app and non-destructive. Your system remains on Wayland.
- If AnyDesk is updated via package manager, the `.desktop` file may be overwritten. Re-apply the `Exec` line fix after updates.
- Tested on AnyDesk 8.0.2 on Ubuntu 26.04 with NVIDIA GPU.
