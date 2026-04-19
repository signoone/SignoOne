# Oh My Zsh — Complete Setup Guide
> Documented from a real Ubuntu installation session
<img width="500" height="304" alt="image" src="https://github.com/user-attachments/assets/2f72b657-2add-4118-8548-420c9725388f" />

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Step 1 — Install Zsh](#step-1--install-zsh)
- [Step 2 — Install Oh My Zsh](#step-2--install-oh-my-zsh)
- [Step 3 — Set Default Shell](#step-3--set-default-shell)
- [Step 4 — Essential Plugins](#step-4--essential-plugins)
- [Step 5 — Built-in Plugins](#step-5--built-in-plugins-no-cloning-needed)
- [Step 6 — Themes](#step-6--themes)
- [Tmux Clipboard Fix](#tmux-clipboard-fix)
- [Quick Reference](#quick-reference)

---

## Prerequisites

Make sure these are installed before anything else:

```bash
sudo apt update
sudo apt install git curl -y
```

> **Lesson learned:** Oh My Zsh installer will fail silently if `git` is missing. Install it first.

---

## Step 1 — Install Zsh

```bash
sudo apt install zsh -y
```

Verify the installation:

```bash
zsh --version
# Expected output: zsh 5.9 (x86_64-ubuntu-linux-gnu)
```

---

## Step 2 — Install Oh My Zsh

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

The installer will:
- Clone Oh My Zsh into `~/.oh-my-zsh`
- Create a default `~/.zshrc` config file
- Ask if you want to set Zsh as your default shell → type `y`

---

## Step 3 — Set Default Shell

The Oh My Zsh installer handles this, but if needed manually:

```bash
chsh -s $(which zsh)
```

> **Important:** Log out and log back in for the change to take effect. Verify with `echo $SHELL` — should return `/usr/bin/zsh`.

---

## Step 4 — Essential Plugins

These require cloning into Oh My Zsh's custom plugins directory.

### zsh-autosuggestions
Suggests commands as you type based on history (grey ghost text). Press `→` to accept.

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

### zsh-completions
Expands tab-completion to cover many more commands and tools.

```bash
git clone https://github.com/zsh-users/zsh-completions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-completions
```

### zsh-syntax-highlighting
Colors commands **green** when valid, **red** when invalid — catches typos before you hit Enter.

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

> ⚠️ **Critical:** `zsh-syntax-highlighting` must always be **last** in the plugins list.

### zsh-history-substring-search
Type part of an old command, press `↑` and it only cycles through matching history entries.

```bash
git clone https://github.com/zsh-users/zsh-history-substring-search \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-history-substring-search
```

### fzf — Fuzzy Finder
Press `Ctrl+R` to get an interactive searchable history list instead of cycling one by one.

```bash
sudo apt install fzf -y
```

---

## Step 5 — Built-in Plugins (No Cloning Needed)

These ship with Oh My Zsh — just add them to your plugins list.

| Plugin | What it does |
|---|---|
| `git` | Short aliases: `gst`, `gco`, `gp`, `glog`, etc. Run `alias \| grep git` to see all |
| `z` | Jump to frequent directories by partial name. `z myapp` instead of full `cd` path |
| `sudo` | Press `Esc` twice to prepend `sudo` to your last command |
| `extract` | `extract file.tar.gz` works on any archive format — no more googling tar flags |
| `command-not-found` | Suggests which package to install when a command doesn't exist |
| `colored-man-pages` | Adds color to man pages so they're actually readable |
| `copypath` | Copies current directory path to clipboard instantly |
| `history` | Adds `h` alias for history, `hsi` for searching it |

---

## Enabling All Plugins

Open your config file:

```bash
nano ~/.zshrc
```

Find the plugins line and update it:

```bash
plugins=(
  git
  z
  sudo
  extract
  command-not-found
  colored-man-pages
  fzf
  zsh-autosuggestions
  zsh-completions
  zsh-history-substring-search
  zsh-syntax-highlighting
)
```

Save (`Ctrl+O` → Enter → `Ctrl+X`), then apply:

```bash
source ~/.zshrc
```

---

## Step 6 — Themes

Open `.zshrc` and find:

```bash
ZSH_THEME="robbyrussell"
```

Replace with any theme name. Popular options:

| Theme | Style |
|---|---|
| `robbyrussell` | Default, clean |
| `agnoster` | Powerline-style, shows git branch (requires Powerline font) |
| `af-magic` | Minimal, informative |
| `bira` | Two-line prompt, clean |
| `fino` | Git-aware, colorful |

Install Powerline fonts if using `agnoster`:

```bash
sudo apt install fonts-powerline -y
```

Apply after changing theme:

```bash
source ~/.zshrc
```


## Quick Reference

| Task | Command |
|---|---|
| Reload zsh config | `source ~/.zshrc` |
| Edit zsh config | `nano ~/.zshrc` |
| Edit tmux config | `nano ~/.tmux.conf` |
| Reload tmux config | `tmux source-file ~/.tmux.conf` |
| Check current shell | `echo $SHELL` |
| Check session type | `echo $XDG_SESSION_TYPE` |
| Accept autosuggestion | `→` right arrow key |
| Fuzzy search history | `Ctrl+R` |
| Add sudo to last command | `Esc` `Esc` |
| Extract any archive | `extract filename.ext` |
| Jump to frequent dir | `z dirname` |

---

*Setup completed on Ubuntu 24 with Zsh 5.9 and Oh My Zsh.*
