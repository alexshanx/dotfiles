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

The installer handles chezmoi, Homebrew packages (including the Codex app), Zsh plugins, development tools, Claude Code, managed files, and macOS preferences. When it finishes:

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

Repositories live under `~/Desktop/codes`. Commits there are signed with the SSH
key `~/.ssh/id_ed25519`; upload its public key to GitHub once as a signing key so
commits show as verified:

```bash
gh auth refresh -h github.com -s admin:ssh_signing_key
gh ssh-key add ~/.ssh/id_ed25519.pub --type signing --title "$(hostname -s)"
```

On a work machine, create `~/.config/git/gitlab.local.config` for repositories under `~/Desktop/codes/gitlab`:

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

`.gitleaksignore` lists historical findings that were reviewed and judged
harmless, by fingerprint; without it a full-history scan blocks every new branch
or tag. Add an entry only after reading the flagged commit.

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

## 5. Zsh Plugins

Zsh plugins are chezmoi externals declared in
[`home/.chezmoiexternal.toml`](home/.chezmoiexternal.toml). Each one is pinned to
a release tag or commit and verified with a SHA-256 checksum, so a moved tag or a
compromised upstream fails `chezmoi apply` instead of running in every shell.
Oh My Zsh is not installed; only its `git` alias plugin is fetched as a single file.

To upgrade a plugin, change its URL, recompute the checksum, and apply:

```bash
curl -fsSL <url> | shasum -a 256
chezmoi apply ~/.zsh/plugins
```

## 6. Manage Personal AI Skills

`~/.agents/skills` is the only source of truth. Claude receives one compatibility symlink per skill under `~/.claude/skills`.

To manage a newly created local skill:

```bash
chezmoi add ~/.agents/skills/<skill-name>
ln -s ../../.agents/skills/<skill-name> ~/.claude/skills/<skill-name>
chezmoi add ~/.claude/skills/<skill-name>
```

Restart the relevant AI tool after adding or renaming a skill.

## 7. Resolve Local Changes

When chezmoi reports that a destination changed since it was last written:

1. Inspect it with `chezmoi diff <path>`.
2. Keep the local version with `chezmoi re-add <path>`, or restore the managed version with `chezmoi apply <path>`.
3. Use `chezmoi apply --force <path>` only after reviewing that specific target; avoid forcing the entire repository.

## 8. Toolchain Updates

Updates are proposed automatically but applied only after review. Nothing on
the Mac changes without you running a command and confirming the diff, so a
compromised bot or account cannot run code on this machine unattended.

### Renovate Proposals

[Renovate](https://docs.renovatebot.com/) proposes updates to the tools pinned in [`~/.proto/.prototools`](home/dot_proto/dot_prototools). Install the Renovate GitHub App for this repository once; [`renovate.json`](renovate.json) handles the rest. Other repository dependencies, including Homebrew packages and GitHub Actions, stay outside this automation.

Renovate checks during the first three days of each month (Asia/Shanghai), waits 14 days after a release, and groups the proto-managed tools into one pull request. It never merges: review the versions and changelogs, wait for CI, and merge the pull request yourself. Node.js stays on its configured major release line so that an odd-numbered, non-LTS release is not selected automatically.

The Mend-hosted app puts repositories into **Silent** mode when it was installed for all repositories: Renovate still scans on schedule and lists updates in the [Mend developer portal](https://developer.mend.io/github/alexshanx/dotfiles), but creates no Dependency Dashboard issue, branches, or pull requests. `"mode": "full"` in `renovate.json` overrides that portal default from the repository itself. Renovate then keeps a **Dependency Dashboard** issue; tick an update there, or use the portal's **Create/Rebase**, to open its pull request outside the schedule.

The Renovate app has write access to this repository regardless of these settings. Protect `main` with a repository ruleset (**Settings → Rules → Rulesets**) that requires a pull request and the `lint` and `apply` checks and blocks force pushes.

### Applying Updates on the Mac

Run [`dotfiles-update`](home/dot_local/bin/executable_dotfiles-update) from a terminal, for example after merging a Renovate pull request:

```bash
dotfiles-update
```

1. Pulls this repository with `--ff-only`, lists the incoming commits, and shows `chezmoi diff`, including the scripts that apply would run. Changes are applied only after you answer `y`.
2. Shows `proto status` and the latest proto release, then, after you answer `y`, runs `proto upgrade` and installs every tool pinned in `~/.proto/.prototools`. A separate prompt offers `proto clean`, which removes tools unused for 30 days. `proto upgrade` installs the latest proto release immediately; the 14-day delay applies only to Renovate-managed tools.
3. Runs `brew update`, lists outdated formulae, and after you answer `y` runs `brew upgrade --formula` and `brew cleanup`. Homebrew no longer builds bottles for Intel Macs, so there an upgrade may compile formulae and new dependencies from source for hours. Casks are left to their own updaters, because cask upgrades may need a password or a running app to quit.

Nothing is upgraded without a `y` answer. Each step runs even if an earlier one fails, and failures are summarized at the end.

## Repository Map

| Path | Purpose |
| --- | --- |
| [install.sh](install.sh) | New-machine entry point |
| [Brewfile](Brewfile) | Homebrew packages and applications |
| [home](home) | Files mapped into the home directory |
| [home/.chezmoiscripts](home/.chezmoiscripts) | Bootstrap and lifecycle automation |
| [renovate.json](renovate.json) | Monthly proto toolchain update proposals |
| [home/dot_local/bin/executable_dotfiles-update](home/dot_local/bin/executable_dotfiles-update) | Reviewed local update of dotfiles, proto, and Homebrew |

Useful inspection commands: `chezmoi status`, `chezmoi diff`, `chezmoi managed`, `chezmoi data`, and `chezmoi cd`.

GitHub Actions scans the full history with Gitleaks, checks shell scripts and Zsh syntax, and runs a Linux apply/verify dry-run. Licensed under the [MIT License](LICENSE).
