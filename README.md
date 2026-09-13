# Dotfiles

Personal macOS setup managed with [chezmoi](https://www.chezmoi.io/). It keeps shell, Git, SSH, applications, development tools, macOS preferences, and personal AI skills reproducible across machines.

## 1. Install a New Mac

You need macOS, an internet connection, and an administrator account for Homebrew.

```bash
curl -fsLS https://raw.githubusercontent.com/alexshanx/dotfiles/main/install.sh | sh
```

During setup:

- Enter the Git email for this machine and choose whether it is a work machine.
- Approve the Xcode Command Line Tools dialog if it appears.
- Enter your administrator password if Homebrew requests it.

The installer handles chezmoi, Homebrew packages (including the Codex app), shell plugins, development tools, Claude Code, managed files, and macOS preferences. When it finishes:

```bash
exec zsh
chezmoi status
```

SSH keys, GPG keys, npm authentication, and other credentials are intentionally left for manual setup.

## 2. Sync an Existing Mac

Pull the latest repository version and apply it:

```bash
chezmoi update
```

To review remote changes before applying them:

```bash
chezmoi git -- pull --ff-only
chezmoi diff
chezmoi apply
```

An unrestricted apply may run lifecycle scripts. A [Brewfile](Brewfile) change, for example, triggers package installation. When only one file needs updating, apply that target directly:

```bash
chezmoi apply ~/.zshrc
```

## 3. Change and Publish Dotfiles

Edit a managed file, review the generated change, and apply it locally:

```bash
chezmoi edit ~/.zshrc
chezmoi diff ~/.zshrc
chezmoi apply ~/.zshrc
```

Add a new file, or capture an intentional change made directly to an existing target:

```bash
chezmoi add ~/.config/example/config.toml
chezmoi re-add ~/.zshrc
```

Commit from the source repository:

```bash
chezmoi cd
git status
git add <reviewed-source-files>
git diff --cached
gitleaks git --pre-commit --staged --redact .
git commit -m "chore: update dotfiles"
git push
exit
```

## 4. Keep Machine-Specific Data Local

The initial Git email and work-machine choice are stored in `~/.config/chezmoi/chezmoi.toml`; inspect them with `chezmoi data`.

On a work machine, create `~/.config/git/gitlab.local.config` for repositories under `~/Code/GitLab`:

```ini
[user]
  email = you@company.com
```

Put secrets and machine-only environment variables in `~/.zshrc.local`. It is loaded automatically and ignored by chezmoi:

```bash
export ANTHROPIC_AUTH_TOKEN="your-token"
export WORK_SPECIFIC_VAR="value"
```

Codex configuration and Claude settings, credentials, sessions, and runtime data
stay local. Claude's `CLAUDE.md` and skill links are managed normally. Two native
chezmoi `create_` files provide initial preferences when configuration is missing:

- Claude: automatic theme, agent and input notifications enabled, and empty
  commit/PR attribution.
- Codex: `commit_attribution = ""`.

Existing file contents are left unchanged, even if they contain different
preferences. Missing files are created with mode `0600`; no merge script or
Python/TOML dependency is needed. Later changes to these defaults affect only
machines where the files do not exist. To initialize just these targets:

```bash
chezmoi apply --parent-dirs ~/.claude/settings.json ~/.codex/config.toml
```

Do not use `chezmoi add` or `re-add` on these configuration files: initialization
does not make their later contents safe to publish. Edit the two source defaults
directly. Git ignores ordinary configuration source files and allows only the
two `create_private_` initializers. `re-add` leaves these create-only defaults
unchanged. Gitleaks permits only their approved default content and blocks
ordinary configuration files if they are forcibly staged.
Intentional default changes require reviewing and updating the matching
`initializer-content` allowlist in `.gitleaks.toml`. Other files are still subject
to the broader secret/privacy rules, which cannot detect every kind of PII.

`.chezmoiignore` filters destination paths; `.gitignore` filters source paths.
Neither is a security boundary: forced adds and already tracked files can bypass
ignores. Never enable automatic dotfile commits or pushes. Review each staged
diff and resolve scanner findings before committing.

### Commit and Push Protection

Homebrew installs Gitleaks. On macOS, chezmoi enables this repository's native
Git hooks during apply. To enable them manually on an existing clone:

```bash
brew install gitleaks
chmod +x .githooks/pre-commit .githooks/pre-push
git config --local core.hooksPath .githooks
git config --get core.hooksPath
gitleaks dir --config .gitleaks.toml --redact .
```

Review any existing `core.hooksPath` before replacing it. Hook installation is
local to each clone; it does not propagate through Git.

The commit hook scans staged additions. The push hook scans outgoing commits
for every ref, including tags. For a new ref or an unavailable remote object it
scans the full reachable history, which may block on historical findings. Remote
ref deletions skip scanning. Missing Gitleaks or a scanner error blocks the
operation. Hooks can be bypassed with Git options or configuration changes;
they do not constrain an agent allowed to disable them.

`.gitleaks.toml` extends the default secret rules with personal absolute home
paths and local-only credential/application files. These rules cannot detect all
personal information or arbitrary prose. Reports redact detected values; do not
publish unredacted reports. Review findings rather than adding broad exceptions.

Enable GitHub's server-side protection separately:

1. Repository **Settings → Advanced Security → Secret Protection**: enable
   **Secret Protection**, then **Push protection**.
2. Personal **Settings → Code security**: ensure **Push protection for yourself**
   is enabled.

See [repository push protection](https://docs.github.com/en/code-security/how-tos/secure-your-secrets/prevent-future-leaks/enable-push-protection)
and [personal push protection](https://docs.github.com/en/code-security/how-tos/secure-your-secrets/prevent-future-leaks/manage-user-push-protection).
GitHub blocks supported secret patterns; these settings do not upload this
repository's custom privacy rules to GitHub. CI runs after upload and cannot
prevent the initial disclosure of a secret.

## 5. Manage Personal AI Skills

`~/.agents/skills` is the only source of truth. Claude receives one compatibility symlink per skill under `~/.claude/skills`.

To manage a newly created local skill:

```bash
chezmoi add ~/.agents/skills/<skill-name>
ln -s ../../.agents/skills/<skill-name> ~/.claude/skills/<skill-name>
chezmoi add ~/.claude/skills/<skill-name>
```

Restart the relevant AI tool after adding or renaming a skill.

## 6. Resolve Local Changes

When chezmoi reports that a destination changed since it was last written:

1. Inspect it with `chezmoi diff <path>`.
2. Keep the local version with `chezmoi re-add <path>`, or restore the managed version with `chezmoi apply <path>`.
3. Use `chezmoi apply --force <path>` only after reviewing that specific target; avoid forcing the entire repository.

## 7. Automated Toolchain Updates

[Renovate](https://docs.renovatebot.com/) updates the tools pinned in [`~/.proto/.prototools`](home/dot_proto/dot_prototools). Install the Renovate GitHub App for this repository once; [`renovate.json`](renovate.json) handles the rest. Other repository dependencies, including Homebrew packages and GitHub Actions, stay outside this automation.

Renovate checks on the first day of each month, waits 14 days after a release, groups the proto-managed tools into one pull request, and merges it after CI passes. Node.js stays on its configured major release line so that an odd-numbered, non-LTS release is not selected automatically. Failed CI leaves the pull request open for investigation.

After an automated update is merged, apply it locally with the normal sync command:

```bash
chezmoi update
```

## Repository Map

| Path | Purpose |
| --- | --- |
| [install.sh](install.sh) | New-machine entry point |
| [Brewfile](Brewfile) | Homebrew packages and applications |
| [home](home) | Files mapped into the home directory |
| [home/.chezmoiscripts](home/.chezmoiscripts) | Bootstrap and lifecycle automation |
| [renovate.json](renovate.json) | Monthly proto toolchain updates |

Useful inspection commands: `chezmoi status`, `chezmoi diff`, `chezmoi managed`, `chezmoi data`, and `chezmoi cd`.

GitHub Actions checks shell scripts, Zsh syntax, and a Linux apply/verify dry-run. Licensed under the [MIT License](LICENSE).
