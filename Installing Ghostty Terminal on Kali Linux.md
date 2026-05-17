# Installing Ghostty Terminal on Kali Linux

## Prerequisites

Install required dependencies:

```bash
sudo apt update
sudo apt install libgtk-4-dev libadwaita-1-dev blueprint-compiler gettext libgtk4-layer-shell-dev
```

---

## 1. Install Zig

Check the required Zig version first (do this after cloning in step 2):

```bash
cat ~/ghostty/build.zig.zon | grep -i minimum
```

Download and install from [ziglang.org/download](https://ziglang.org/download/):

```bash
cd ~
wget https://ziglang.org/download/0.15.2/zig-x86_64-linux-0.15.2.tar.xz
tar -xf zig-x86_64-linux-0.15.2.tar.xz
sudo mv zig-x86_64-linux-0.15.2 /opt/zig
sudo ln -s /opt/zig/zig /usr/local/bin/zig
zig version
```

---

## 2. Clone Ghostty

```bash
git clone https://github.com/ghostty-org/ghostty
cd ghostty
```

---

## 3. Build Ghostty

```bash
zig build -Doptimize=ReleaseFast
```

This takes a while. Wait until it returns to the prompt with no errors.

---

## 4. Install System-wide

```bash
sudo zig build -p /usr -Doptimize=ReleaseFast
```

---

## 5. Launch

```bash
ghostty
```

> Ghostty will not appear in the app launcher menu immediately after install. Log out and log back in for it to show up.

---

## 6. Font Configuration (optional)

```bash
mkdir -p ~/.config/ghostty
nano ~/.config/ghostty/config
```

Add:

```
font-family = JetBrains Mono
font-size = 13
```

Reload config inside Ghostty: `Shift+Ctrl+,`

To find available font names:

```bash
fc-list | grep -i "font name"
```

---

## 7. Themes

**Browse built-in themes (interactive preview):**

```bash
ghostty +list-themes
```

- Use arrow keys to scroll, `/` to search, `c` to copy theme name, `Ctrl+C` to exit.

**Apply a theme:**

```bash
nano ~/.config/ghostty/config
```

Add:

```
theme = Catppuccin Frappe
```

> Use quotes if the theme name has spaces: `theme = "Monokai Classic"`

Reload config: `Shift+Ctrl+,`

**Separate light/dark themes:**

```
theme = dark:Catppuccin Frappe,light:Catppuccin Latte
```

Ghostty switches automatically based on system appearance.

**Custom themes:**

Place a custom theme file in `~/.config/ghostty/themes/` and reference it by filename:

```
theme = mytheme
```

---

## Key Shortcuts

| Action | Shortcut |
|---|---|
| New Tab | `Shift+Ctrl+T` |
| New Window | `Shift+Ctrl+N` |
| Close Tab | `Shift+Ctrl+W` |
| Command Palette | `Shift+Ctrl+P` |
| Reload Config | `Shift+Ctrl+,` |
| Quit | `Shift+Ctrl+Q` |
