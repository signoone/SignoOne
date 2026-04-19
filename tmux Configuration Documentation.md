# tmux Configuration Documentation

<img width="305" height="80" alt="image" src="https://github.com/user-attachments/assets/ed5690be-36ac-478e-9381-310f94d02129" />







**File:** `.tmux.conf`  
**Shell:** Intended for Linux environments with `xclip` available (X11).

---

## Full Configuration (Copy-Paste Ready)
Copy and Paste in your Home Directory and name it as `.tmux.conf`  

```bash
set -g mouse on
set -g default-terminal "screen-256color"
bind -n WheelUpPane if-shell -F -t = "#{mouse_any_flag}" "send-keys -M" "if -Ft= '#{pane_in_mode}' 'send-keys -M' 'select-pane -t=; copy-mode -e; send-keys -M'"
bind -n WheelDownPane select-pane -t= \; send-keys -M
bind -n C-WheelUpPane select-pane -t= \; copy-mode -e \; send-keys -M
bind -T copy-mode-vi    C-WheelUpPane   send-keys -X halfpage-up
bind -T copy-mode-vi    C-WheelDownPane send-keys -X halfpage-down
bind -T copy-mode-emacs C-WheelUpPane   send-keys -X halfpage-up
bind -T copy-mode-emacs C-WheelDownPane send-keys -X halfpage-down
bind right split-window -h -c "#{pane_current_path}"
bind down split-window -v -c "#{pane_current_path}"


bind c new-window -c "#{pane_current_path}"

set-option -g history-limit 99999999


# To copy, left click and drag to highlight text in yellow,
# once you release left click yellow text will disappear and will automatically be available in clipboard
# Use vim keybindings in copy mode
setw -g mode-keys vi
# Update default binding of `Enter` to also use copy-pipe
unbind -T copy-mode-vi Enter
bind-key -T copy-mode-vi Enter send-keys -X copy-pipe-and-cancel "xclip -selection c"
bind-key -T copy-mode-vi MouseDragEnd1Pane send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"

bind-key / copy-mode \; send-key ?
```

---

## Overview

This config does four things:
1. Enables mouse support with sane scroll behavior
2. Sets up intuitive pane/window splitting
3. Configures vi-style copy mode with system clipboard integration via `xclip`
4. Bumps history to effectively unlimited

---

## Settings

### Terminal & Mouse

| Setting | Value | Effect |
|---|---|---|
| `mouse` | `on` | Enables full mouse support — click to focus panes, drag to resize, scroll to navigate |
| `default-terminal` | `screen-256color` | Declares 256-color support to the terminal; required for proper color rendering in vim, etc. |
| `history-limit` | `99999999` | Effectively unlimited scrollback buffer per pane |

---

### Mouse Scroll Behavior

These bindings refine how scroll events behave depending on context:

```
bind -n WheelUpPane   → Smart scroll: sends scroll to app if it handles mouse, else enters copy mode
bind -n WheelDownPane → Focuses pane and passes scroll event through
bind -n C-WheelUpPane → Ctrl+scroll: focuses pane, enters copy mode
```

**In copy mode** (both vi and emacs key tables):

| Binding | Action |
|---|---|
| `Ctrl + ScrollUp` | Half-page up |
| `Ctrl + ScrollDown` | Half-page down |

---

### Pane Splitting

Both bindings inherit the **current pane's working directory** — no more landing in `$HOME` on every split.

| Binding | Action |
|---|---|
| `Prefix + →` (Right arrow) | Split pane **horizontally** (side by side) |
| `Prefix + ↓` (Down arrow) | Split pane **vertically** (top and bottom) |

> **Note:** Default tmux prefix is `Ctrl+b` unless overridden elsewhere.

---

### Window Creation

| Binding | Action |
|---|---|
| `Prefix + c` | New window, opens in the **current pane's directory** |

This overrides tmux's default `new-window` behavior which opens in the shell's `$HOME`.

---

### Copy Mode

**Key table:** `vi`

```
setw -g mode-keys vi
```

Copy mode uses vim keybindings (`v` to select, `y` to yank, `/` to search, etc.).

#### Clipboard Integration

Copies go directly to the **system clipboard** via `xclip`. Requires `xclip` to be installed and an active X11 session.

| Binding | Trigger | Action |
|---|---|---|
| `Enter` (in copy mode) | Keyboard selection confirm | Copies selection to clipboard, exits copy mode |
| `MouseDragEnd1Pane` | Release left mouse button after drag | Copies highlighted text to clipboard, exits copy mode |

> The default `Enter` binding in vi copy mode is unbound and replaced to ensure clipboard pipe behavior.

#### Mouse Copy Workflow

1. Left-click and drag to highlight text — selection highlights in yellow
2. Release the mouse button
3. Highlight disappears — text is now in your system clipboard
4. Paste anywhere with `Ctrl+V` or middle-click

---

### Search Shortcut

| Binding | Action |
|---|---|
| `Prefix + /` | Enters copy mode and immediately opens reverse search (`?`) |

This is a quality-of-life shortcut. In vi copy mode, `?` searches backward through the buffer — useful for finding recent output without manually entering copy mode first.

---

## Dependencies

| Dependency | Why |
|---|---|
| `xclip` | Required for clipboard integration. Install via `sudo apt install xclip` (Debian/Ubuntu) or equivalent. |
| X11 session | `xclip` won't work in a headless/SSH session without X forwarding. Use `xclip` alternatives like `xsel` or swap for `wl-copy` on Wayland. |

---

## Compatibility Notes

- **Wayland users:** Replace `xclip -selection c` and `xclip -in -selection clipboard` with `wl-copy` in both `copy-pipe-and-cancel` calls.
- **SSH without X forwarding:** Clipboard piping will silently fail. Consider `tmux-yank` plugin or `pbcopy` on macOS.
- **tmux version:** `copy-pipe-and-cancel` requires tmux ≥ 2.4. Mouse bind syntax requires tmux ≥ 2.1.
