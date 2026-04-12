# Article Notes: Building a Reproducible Linux Dev Environment

Raw notes per iteration — raw material for a Medium article.

Target audience: developers who want a modern, reproducible dev environment that works on any Linux machine (OrbStack VMs, Azure VMs, WSL) with a single command.

---

## Iteration 1 — Bare minimum bootstrap

### What was built
- chezmoi as the dotfile manager
- A `run_once_` script that installs curl, git, zsh and sets zsh as the default shell
- A minimal `.zshrc` placeholder
- `.chezmoiignore` to keep repo meta-files out of `~`

### Why chezmoi
Most dotfile managers are just symlink farms. chezmoi treats dotfiles as source-controlled templates with:
- Per-machine templating (different config on different machines)
- `run_once_` scripts that bootstrap the system idempotently — run on first apply, never again
- A single one-liner to fully restore an environment from scratch

### Why start this small
The most common mistake when building a dev environment setup is trying to do everything at once, then never being sure which part is broken. Starting with just "does zsh get installed and set as default?" proves the whole pipeline works before adding complexity.

### The one-liner
```
sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply evgenii-prusov
```
One command. Clean machine. Full environment. That's the goal.

### Friction points during setup
Two issues hit on the first real SSH connection:

1. **`chsh` fails non-interactively** — `chsh` uses PAM authentication which requires a TTY. The fix is `sudo usermod -s $(which zsh) $USER`, which doesn't need a password prompt. Worth knowing if you ever script shell changes.

2. **Broken backspace via SSH** — Ghostty (and other modern terminals) forward `TERM=xterm-ghostty` over SSH, but the remote VM has no terminfo entry for it. Without a valid TERM, the terminal is essentially blind — key bindings don't work, rendering breaks. The fix: check if the terminfo entry exists and fall back to `xterm-256color` if not:
   ```zsh
   if [[ -z "$TERM" ]] || ! infocmp "$TERM" &>/dev/null; then
     export TERM=xterm-256color
   fi
   ```
   This is a good general pattern for any server-side zshrc — you can't control what TERM clients send.

### Relevance to AI coding tools
Modern AI coding tools (Claude Code, Cursor, Aider, etc.) all run in the terminal. A reproducible environment means you can spin up a fresh VM for an AI coding session, have your full toolchain in minutes, and tear it down when done. No manual setup, no "it works on my machine."

---
