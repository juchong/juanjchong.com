+++
title = "Connecting Codex to TrueNAS over SSH"
date = "2026-09-15"
author = "Juan Chong"
tags = ["Guides", "TrueNAS", "Docker", "AI"]
description = "Installing Codex in a TrueNAS user's home directory and connecting the desktop app to Docker Compose projects over SSH"
draft = true
Toc = true
+++

I run Docker Compose stacks on TrueNAS and wanted to manage them through Codex's
desktop app. The project files and logs are on the NAS, along with Docker and the
other tools I need. Copying the files to my Mac would not give Codex access to the
system running them.

SSH worked, but the desktop app reported:

```text
Codex CLI not installed
Codex CLI is not installed on this remote machine
```

The missing component was the remote Codex executable. I installed OpenAI's
standalone Linux package in my user's persistent home directory, made `codex`
available in the login shell, and signed in. The desktop app then connected to
the project on the NAS.

I tested this on **TrueNAS 25.10.6, x86-64, with Codex CLI 0.154.0**. This is a
user-managed installation, not a TrueNAS-supported package.

## Requirements

The desktop app uses SSH to start a Codex app server on the remote host.
[OpenAI's SSH documentation](https://learn.chatgpt.com/docs/remote-connections#connect-to-an-ssh-host)
requires `codex` to be installed, authenticated, and available in the remote
user's login-shell `PATH`.

TrueNAS protects its OS filesystem and disables tools such as `apt`. Leave those
protections in place.
[Developer mode is explicitly discouraged on production storage systems](https://www.truenas.com/docs/scale/25.10/scaletutorials/systemsettings/advanced/developermode/),
and this installation does not need it. It also does not require Node.js or npm.

Use an SSH account with a writable home directory on a persistent data pool.
The account must be able to execute files there.

## Check SSH and the home directory

The examples use `nas` as the SSH alias, `devuser` as the remote account, and
`/mnt/tank/...` for dataset paths. Substitute your own values.

On the Mac, configure the host in `~/.ssh/config` using your existing SSH key:

```sshconfig
Host nas
    HostName nas.example.com
    User devuser
    IdentityFile ~/.ssh/id_ed25519
```

Connect:

```bash
ssh nas
```

On TrueNAS, check the platform and home directory:

```bash
uname -sm
printf '%s\n' "$HOME"
```

These instructions use the Linux x86-64 package. Your home directory should be
on a data pool, such as `/mnt/tank/users/devuser`. If it is on the boot filesystem,
configure a persistent home before installing.

## Download and test the package

[OpenAI distributes a standalone CLI for Linux](https://learn.chatgpt.com/docs/codex/cli).
I downloaded the archive directly, checked its checksum, and tested the executable
before adding it to `PATH`. The commands below use version 0.154.0.

Run these blocks in the **same Bash SSH session**; later commands reuse the
variables. `set -euo pipefail` stops on errors. If a check fails, fix it before
continuing.

```bash
set -euo pipefail
umask 077

test "$(uname -s)" = Linux
test "$(uname -m)" = x86_64

codex_version=0.154.0
codex_target=x86_64-unknown-linux-musl
codex_archive="codex-package-${codex_target}.tar.gz"
codex_source="https://releases.openai.com/codex/releases/${codex_version}"
codex_stage=$(mktemp -d "$HOME/codex-stage.XXXXXXXX")

curl --fail --show-error --location \
    --proto '=https' --proto-redir '=https' \
    "${codex_source}/${codex_archive}" \
    -o "${codex_stage}/${codex_archive}"

codex_sha256=fc6e3e3b85f2cf7d664520ee5c66a7fe4aa12bae7d46834f47e2f165fd0d6f78
printf '%s  %s\n' "$codex_sha256" "${codex_stage}/${codex_archive}" \
    | sha256sum --check -
```

The SHA-256 above matched OpenAI's release metadata and
[checksum manifest for 0.154.0](https://releases.openai.com/codex/releases/0.154.0/codex-package_SHA256SUMS)
at the time of installation. Use the matching published checksum if you choose a
different version or architecture.

Inspect the archive, extract it, and run the version check:

```bash
tar -tvzf "${codex_stage}/${codex_archive}"

mkdir "${codex_stage}/package"
tar --extract --gzip \
    --file "${codex_stage}/${codex_archive}" \
    --directory "${codex_stage}/package" \
    --no-same-owner --no-same-permissions --keep-old-files

"${codex_stage}/package/bin/codex" --version
```

The version check returned:

```text
codex-cli 0.154.0
```

Keep the extracted package together. It includes the code-mode host, ripgrep,
bubblewrap, and zsh in addition to the main executable.

## Install in the user directory

I followed the standalone installer's directory layout: a versioned release under
`~/.codex/packages/standalone/releases/`, a `current` symlink pointing to that
release, and `~/.local/bin/codex` pointing to `current/bin/codex`.

This block is for a **fresh installation**. It stops if the release, `current`
link, or `codex` command already exists. Check those paths before replacing
anything. The parent directories must be within your persistent home, not
symlinks into the system filesystem.

```bash
codex_root="$HOME/.codex/packages/standalone"
codex_release="${codex_root}/releases/${codex_version}-${codex_target}"
codex_bin="$HOME/.local/bin"

for codex_existing in "$codex_release" "$codex_root/current" "$codex_bin/codex"; do
    if [ -e "$codex_existing" ] || [ -L "$codex_existing" ]; then
        printf 'Existing path; inspect before proceeding: %s\n' "$codex_existing" >&2
        exit 1
    fi
done

mkdir -p "$codex_root/releases" "$codex_bin"
mkdir "$codex_release"
tar --extract --gzip \
    --file "${codex_stage}/${codex_archive}" \
    --directory "$codex_release" \
    --no-same-owner --no-same-permissions --keep-old-files

ln -s bin/codex "$codex_release/codex"
"$codex_release/bin/codex" --version

ln -s "$codex_release" "$codex_root/current"
ln -s "$codex_root/current/bin/codex" "$codex_bin/codex"
```

### Make codex available to the login shell

My account's existing `~/.profile` already contained this block:

```bash
if [ -d "$HOME/.local/bin" ]; then
    PATH="$HOME/.local/bin:$PATH"
fi
```

Creating `.local/bin` was enough for the existing profile to add it to `PATH` on
the next login. To check that the login shell can find `codex`, run this on the Mac:

```bash
ssh nas 'bash -lc "command -v codex && codex --version"'
```

With the example home directory, the output should be:

```text
/mnt/tank/users/devuser/.local/bin/codex
codex-cli 0.154.0
```

If the first line is missing, inspect the account's login profile.
[Bash reads the first available login file](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)
among `~/.bash_profile`, `~/.bash_login`, and `~/.profile`. If `.bash_profile`
exists, an edit to `.profile` may never be read. Add `~/.local/bin` to `PATH` in
the profile your account uses, then repeat the check.

## Sign in from a headless server

I used
[OpenAI's device-code flow](https://learn.chatgpt.com/docs/auth#login-on-headless-devices).
The login command runs on TrueNAS, and I approve it in the browser on my Mac.

First enable device-code authorization for Codex in
[ChatGPT Security Settings](https://chatgpt.com/#settings/Security). Managed
workspaces may require an administrator to enable it in workspace permissions.
Then run this from the Mac:

```bash
ssh nas 'bash -lc "codex login --device-auth"'
```

Open the printed link in your browser, enter the code from your login command,
and approve the sign-in. Leave the SSH command running until it reports success.

I missed the security setting on the first attempt. Enabling it was enough for
the original code to work. If yours has expired, run the login command again.

Check the login status:

```bash
ssh nas 'bash -lc "codex login status"'
```

Mine returned:

```text
Logged in using ChatGPT
```

Codex saves the login for subsequent sessions. If it uses `~/.codex/auth.json`,
that file contains credentials. Keep it out of the stack repository.

## Connect the desktop app

In the desktop app, open **Settings → Connections → SSH**, reconnect the host, and
select the project directory on the NAS, such as `/mnt/tank/docker/stacks`.

At this point, my desktop app connected successfully. The files stayed on TrueNAS,
and I could select the remote project from the app.

File permissions, Docker socket access, and `sudo` policy still apply to that SSH
account. I verified the executable, login-shell command lookup, authentication,
and desktop connection for this post. Sandbox behavior and stack operations need
their own testing.
