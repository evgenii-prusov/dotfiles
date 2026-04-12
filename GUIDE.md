# Building a Reproducible Linux Dev Environment with Chezmoi

A step-by-step guide to setting up a modern, portable developer environment that bootstraps from a single command on any Linux machine — OrbStack VMs, Azure VMs, WSL, you name it.

---

## Prerequisites

> **TODO:** Add setup instructions for OrbStack and creating a GitHub repo for dotfiles.

---

## Iteration 1 — Getting chezmoi to bootstrap a machine

Before adding any real tooling, we need to prove the pipeline works. The goal of this first step is simple: run one command on a fresh machine and have it install zsh and set it as your default shell. Nothing more.

### Create the project structure

In your dotfiles repo, create three files.

**`.chezmoiignore`** — tells chezmoi which files in the repo to ignore when deploying to `~`. We don't want PLAN.md or this guide ending up in your home directory:

```
PLAN.md
GUIDE.md
README.md
.git
bin/
```

**`run_once_01-install-packages.sh.tmpl`** — a script chezmoi runs exactly once on first apply. The `run_once_` prefix is chezmoi's way of saying "run this, then never again" (it tracks a hash of the file). The `01-` prefix controls execution order when you add more scripts later:

```bash
#!/bin/bash
set -euo pipefail

sudo apt-get update -q
sudo apt-get install -y curl git zsh

if [ "$SHELL" != "$(which zsh)" ]; then
  sudo usermod -s "$(which zsh)" "$USER"
fi
```

Note: we use `usermod` instead of `chsh` because `chsh` requires PAM authentication, which fails in non-interactive (scripted) environments.

**`dot_zshrc`** — chezmoi deploys this as `~/.zshrc`. For now, just a minimal config that fixes a common SSH issue: modern terminals like Ghostty forward `TERM=xterm-ghostty`, but remote machines don't have that terminfo entry. Without it, your backspace key types spaces instead of deleting. The fix — check if the terminfo exists and fall back to `xterm-256color`:

```zsh
# ~/.zshrc

# Fall back to xterm-256color if the TERM has no terminfo entry on this machine
if [[ -z "$TERM" ]] || ! infocmp "$TERM" &>/dev/null; then
  export TERM=xterm-256color
fi

# Key bindings
bindkey -e
bindkey "^?" backward-delete-char
bindkey "^H" backward-delete-char
bindkey "^[[3~" delete-char
bindkey "^[[H" beginning-of-line
bindkey "^[[F" end-of-line
```

### Test on a fresh machine

Spin up a clean Ubuntu VM in OrbStack:

```bash
orbctl create ubuntu dotfiles-test
```

Copy your local repo into it:

```bash
orb push -m dotfiles-test /path/to/your/dotfiles dotfiles
```

SSH in and run chezmoi:

```bash
ssh orb
sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply --source ~/dotfiles
```

Verify zsh was set as the default shell:

```bash
grep $USER /etc/passwd
# Should end in: /usr/bin/zsh
```

Open a new shell and confirm backspace works.

### Commit

Once verified, commit and push:

```bash
git add .chezmoiignore dot_zshrc run_once_01-install-packages.sh.tmpl
git commit -m "iteration 1: bare minimum bootstrap (zsh + chezmoi)"
git push
```

---

*More iterations coming — next up: Homebrew and a proper prompt.*
