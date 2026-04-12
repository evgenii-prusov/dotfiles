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

## Iteration 2 — Homebrew, Starship, and gh CLI

With the pipeline proven, we can now add real tooling. This iteration installs Linuxbrew, sets up the Starship prompt, and adds the GitHub CLI — which we'll use in the next step to clone repos and test the prompt across different project types.

### Add the bootstrap script

Create `run_once_02-install-brew-and-tools.sh.tmpl`. The `02-` prefix ensures it runs after the apt script:

```bash
#!/bin/bash
set -euo pipefail

# Install Linuxbrew if not already installed
if ! command -v brew &>/dev/null; then
  NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
fi

# Add brew to PATH for the rest of this script
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"

# Install tools
brew install starship gh
```

Two things worth noting:

- `NONINTERACTIVE=1` skips all prompts in the Homebrew installer — essential for scripted use.
- Brew is added to `PATH` immediately after install so subsequent `brew install` calls work within the same script run.

### Update dot_zshrc

Add brew and starship initialization after the key-binding section:

```zsh
# Homebrew (Linux)
if [[ -x /home/linuxbrew/.linuxbrew/bin/brew ]]; then
  eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
fi

# Starship prompt
if command -v starship &>/dev/null; then
  eval "$(starship init zsh)"
fi
```

Both blocks are guarded so `.zshrc` doesn't error on machines that haven't run iteration 2 yet.

### Configure SSH to forward the token

gh CLI reads `GITHUB_TOKEN` from the environment automatically — no `gh auth login` needed. You just need the variable forwarded when you SSH in.

Add a `Host orb` block to your Mac's `~/.ssh/config`, **before** the OrbStack `Include` line (SSH uses first-match, and OrbStack's include already defines the `orb` host — the block below only adds `SendEnv` to it):

```
Host orb
  SendEnv GITHUB_TOKEN
```

With `GITHUB_TOKEN` set on your Mac, every `ssh dotfiles-test@orb` session will have it available automatically, and `gh` will work without any explicit login step.

### Test on a fresh machine

```bash
orbctl create ubuntu dotfiles-test
orb push -m dotfiles-test ~/projects/dotfiles dotfiles
```

Bootstrap chezmoi and apply:

```bash
curl -fsLS get.chezmoi.io | ssh dotfiles-test@orb "GITHUB_TOKEN=$GITHUB_TOKEN sh -s -- init --apply --source ~/dotfiles"
```

The naive form — `ssh … "sh -c '$(curl …)'"` — doesn't work because `$(curl …)` expands on the Mac before the command is sent, embedding the entire install script as a literal string in the SSH invocation and breaking its internal quoting. Piping curl's output to `sh -s` on the remote avoids this entirely: curl runs on the Mac, the script runs on the VM, and `-- init --apply --source ~/dotfiles` are passed as arguments to it.

Verify:

```bash
ssh dotfiles-test@orb

# Starship prompt should render
gh auth status   # should show logged in via GITHUB_TOKEN
gh repo list
```

### Commit

```bash
git add run_once_02-install-brew-and-tools.sh.tmpl dot_zshrc
git commit -m "iteration 2: homebrew + starship + gh cli"
```

---

*More iterations coming — next up: shell plugins with Zinit.*
