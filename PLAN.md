# Plan: Linux Dotfiles Setup with Chezmoi

## Context
Building a portable, automated, reproducible Linux dev environment from scratch using chezmoi. Must bootstrap with a single command across OrbStack Ubuntu VMs, Azure VMs, and WSL. The work will be done inside a fresh OrbStack Ubuntu VM.

## Pre-requisites (on macOS host)
1. **Create a fresh Ubuntu VM** in OrbStack
2. **Create a GitHub repo** `dotfiles` (empty, no README)
3. **SSH into the VM** and do all remaining work there

---

## Step 1: Initialize chezmoi inside the VM
- Install chezmoi: `sh -c "$(curl -fsLS get.chezmoi.io)"`
- Run `chezmoi init` to create `~/.local/share/chezmoi`
- Link to the GitHub repo with `chezmoi init --apply github.com/<user>/dotfiles`

## Step 2: Bootstrap script — `run_once_01-install-packages.sh.tmpl`
The main idempotent install script. Numbering (`01-`) ensures execution order.

**What it does:**
1. Install base apt dependencies (build-essential, curl, git, zsh, etc.)
2. Install Homebrew/Linuxbrew if not present
3. Use `brew install` for modern tools: neovim, tmux, starship, zoxide, fzf, ripgrep, eza, bitwarden-cli
4. Set Zsh as default shell if not already

**Key principles:**
- `set -euo pipefail` at the top
- Guard every install with `command -v <tool> || install`
- Use `NONINTERACTIVE=1` for Homebrew install
- Linuxbrew path setup via eval in the script itself (so brew is available for subsequent commands)

## Step 3: Zsh config — `dot_zshrc`
Minimal, fast Zsh config with Zinit plugin manager.

**Contents:**
- Homebrew path setup (`eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"`)
- Zinit bootstrap (auto-install if missing)
- Plugins via Zinit (turbo-loaded for speed):
  - `zsh-autosuggestions` — inline suggestions from history
  - `zsh-syntax-highlighting` — command coloring
  - `zsh-completions` — extra completions
- Starship prompt init (`eval "$(starship init zsh)"`)
- Zoxide init (`eval "$(zoxide init zsh)"`)
- fzf key bindings + completion
- Useful aliases: `ls` -> `eza`, `cat` -> `bat` (if installed), `grep` -> `rg`
- History config (large history, shared across sessions, no duplicates)

## Step 4: Tmux config — `dot_tmux.conf`
Minimal modern tmux config.

**Contents:**
- Set prefix to `C-a` (more ergonomic than `C-b`)
- Enable mouse support
- Start window/pane numbering at 1
- Vi-style pane navigation (`h/j/k/l`)
- OSC 52 clipboard integration (`set -s set-clipboard on`)
- Increase scrollback buffer
- Sensible split keybindings (`|` for vertical, `-` for horizontal)
- 256-color / true-color support
- Status bar: minimal, clean

## Step 5: Starship prompt — `dot_config/starship.toml`
Minimal Starship config.

**Contents:**
- Clean prompt format: directory + git branch/status + language version (when in project) + command duration + newline + prompt char
- Truncate directory to 3 levels
- Disable unnecessary modules (keep it fast)

## Step 6: Neovim config — `dot_config/nvim/init.lua`
Minimal baseline only (plugins/LSP to be added later).

**Contents:**
- Line numbers (relative)
- Clipboard integration (OSC 52 via `unnamedplus`)
- Sensible defaults: expandtab, shiftwidth=2, ignorecase+smartcase search
- Basic keymaps: leader=space, clear search highlight, window navigation

## Step 7: Git config — `dot_gitconfig.tmpl`
Chezmoi template for portable git config.

**Contents:**
- User name/email (templated via `chezmoi data`)
- Default branch: `main`
- Common aliases (`co`, `br`, `st`, `lg`)
- Pull rebase by default
- Delta or diff-so-fancy as pager (if installed)

## Step 8: Connect to GitHub and verify
- `chezmoi cd` and `git init && git remote add origin ...`
- Commit all files and push
- **Verification test:** Destroy the VM, create a new one, run the one-liner:
  ```
  sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply <github-user>
  ```
  Confirm everything installs and configures correctly.

---

## File tree (inside chezmoi source)
```
~/.local/share/chezmoi/
  .chezmoiignore
  run_once_01-install-packages.sh.tmpl
  dot_zshrc
  dot_tmux.conf
  dot_config/
    starship.toml
    nvim/
      init.lua
  dot_gitconfig.tmpl
```

## Verification
1. Run `chezmoi apply -v` — all files deployed without errors
2. Open new Zsh shell — Zinit installs plugins, prompt renders via Starship
3. Run `tmux` — prefix `C-a` works, mouse works, panes split correctly
4. Run `nvim` — opens with line numbers, leader=space works
5. Copy text in tmux — OSC 52 sends to host clipboard
6. Destroy VM, recreate, run one-liner — full environment restored
