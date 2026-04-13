# dotfiles

Modern Linux dev environment managed with [chezmoi](https://chezmoi.io). Installs zsh, Homebrew, Starship, Zinit plugins, zoxide, fzf, and eza — bootstrapped from a single command.

## Bootstrap a new machine

```bash
sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply evgenii-prusov
```

Clones this repo from GitHub and applies it. No manual file copying.

## Develop the dotfiles

When testing changes that haven't been committed yet, copy the local repo to the VM and apply from it:

```bash
orb push -m dotfiles-test ~/projects/dotfiles dotfiles
curl -fsLS get.chezmoi.io | ssh dotfiles-test@orb "sh -s -- init --apply --source ~/dotfiles"
```

See `GUIDE.md` for the full iteration workflow.
