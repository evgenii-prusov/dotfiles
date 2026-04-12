# Plan: Linux Dotfiles Setup with Chezmoi

## Approach
Iterative. Each iteration adds one small change (a tool, a config tweak), then gets verified on a fresh OrbStack Ubuntu VM. Nothing moves forward until the previous step reproduces cleanly.

## Workflow per iteration
1. Make a change in the dotfiles repo
2. Spin up a fresh OrbStack Ubuntu VM
3. Copy the local repo into the VM and run `chezmoi init --apply` from it
4. Verify the change works
5. Destroy the VM
6. Only after verification passes: commit and push
7. Move to the next iteration

---

## Iteration 1 — Bare minimum bootstrap
- Install chezmoi
- `run_once_01-install-packages.sh.tmpl`: only installs base apt packages (curl, git, zsh) and sets zsh as default shell
- `dot_zshrc`: empty/minimal (just enough to not break)
- `.chezmoiignore`: ignore PLAN.md and repo meta files
- **Verify:** VM has zsh as default shell, chezmoi applied cleanly

## Iteration 2 — Homebrew + one tool
- Add Linuxbrew install to the bootstrap script
- `brew install` one tool (e.g. starship)
- Add starship init to dot_zshrc
- **Verify:** starship prompt renders in zsh

## Iteration 3 — Shell plugins (Zinit)
- Add Zinit bootstrap + plugins to dot_zshrc (autosuggestions, syntax-highlighting, completions)
- **Verify:** plugins load, suggestions and highlighting work

## Iteration 4 — Shell tools (zoxide, fzf, eza)
- Add `brew install zoxide fzf eza`
- Add their init/aliases to dot_zshrc
- **Verify:** `z`, `fzf`, `ls` (eza) all work

## Iteration 5 — Tmux
- Add `brew install tmux`
- Add `dot_tmux.conf` (prefix C-a, mouse, vi nav, OSC 52, true-color)
- **Verify:** tmux starts, prefix works, mouse works, panes split

## Iteration 6 — Neovim
- Add `brew install neovim`
- Add `dot_config/nvim/init.lua` (minimal: line numbers, clipboard, basic keymaps)
- **Verify:** nvim opens cleanly with config applied

## Iteration 7 — Git config
- Add `dot_gitconfig.tmpl` (templated name/email, aliases, pull rebase, delta pager)
- **Verify:** `git config --list` shows expected values

## Iteration 8 — Starship customization
- Add `dot_config/starship.toml` (prompt format, directory truncation, disable noisy modules)
- **Verify:** prompt looks right in a git repo, in a node project, etc.

---

## Future iterations (as needed)
- Neovim plugins / LSP
- bat, ripgrep, delta
- SSH config
- Additional shell aliases/functions
- Machine-specific config via chezmoi templates
