# New machine bootstrap

Use this runbook after a server or a development box can be reached through
Tailscale. It installs the daily tools and signs in to the CLIs without a
browser on the machine.

Written on 2026-11-23 for Debian 13. All names, addresses, and identities below
are examples. Replace them before use.

For the server setup itself, see [VPS setup with Tailscale SSH](vps-tailscale-setup.md).

## Prerequisites

- A non-root Linux account with `sudo`, reached as `ssh admin@<server>`.
- On your client: the 1Password CLI (`op`), signed in.
- Run each step as the non-root account unless the step says `sudo`.

## 1. System packages

A minimal Debian image does not have some tools that installers need. The Bun
installer stops with `unzip is required`.

```bash
sudo apt install -y git curl unzip zip fzf zoxide gh
```

## 2. Node.js 22 or newer

Debian 13 ships Node.js 20. `wrangler` and `cf` need Node.js 22 or newer.
Install the official release for the user only, and check its checksum:

```bash
cd /tmp
V=$(curl -fsSL https://nodejs.org/dist/latest-v24.x/SHASUMS256.txt | grep -o 'node-v24[0-9.]*-linux-x64.tar.xz' | head -1)
curl -fsSLO "https://nodejs.org/dist/latest-v24.x/$V"
curl -fsSL https://nodejs.org/dist/latest-v24.x/SHASUMS256.txt | grep " $V\$" | sha256sum -c -
mkdir -p ~/.local/share/node ~/.local/bin
tar -xJf "$V" -C ~/.local/share/node --strip-components=1
rm -f "$V"
for b in node npm npx corepack; do ln -sf ~/.local/share/node/bin/$b ~/.local/bin/$b; done
node --version
```

`~/.local/bin` must come before `/usr/bin` in `PATH`.

## 3. User tools

```bash
curl -fsSL https://bun.sh/install | bash
curl -LsSf https://astral.sh/uv/install.sh | sh
curl -fsSL https://starship.rs/install.sh | sudo sh -s -- -y
curl -fsSL https://claude.ai/install.sh | bash
export PATH="$HOME/.local/bin:$HOME/.bun/bin:$PATH"
uv tool install aswap
bun add -g cf wrangler
```

Install Codex from the latest GitHub release (`codex-x86_64-unknown-linux-musl.tar.gz`)
into `~/.local/bin`. Install other CLIs with their own install scripts.

Rex: install the package, then start a personal server that keeps running
after logout:

```bash
rex server install
loginctl enable-linger "$USER"
```

## 4. Shell configuration

Add these lines to `~/.bashrc`:

```bash
. /usr/share/doc/fzf/examples/key-bindings.bash
eval "$(starship init bash)"
eval "$(zoxide init bash)"
```

The `zoxide init` line must be the last line of the file. Otherwise `z`
prints `zoxide: detected a possible configuration issue`. Installers such as
Bun and uv add lines at the end. After each installer, move the `zoxide` line
to the end again and remove duplicate blocks:

```bash
grep -v 'zoxide init bash' ~/.bashrc > ~/.bashrc.tmp
echo 'eval "$(zoxide init bash)"' >> ~/.bashrc.tmp
mv ~/.bashrc.tmp ~/.bashrc
grep -n -E 'BUN_INSTALL|zoxide' ~/.bashrc
```

## 5. Sign in without a browser on the machine

| Tool | Method |
| --- | --- |
| `gh` | `gh auth login -h github.com -p https -w`, then enter the device code at `https://github.com/login/device`. Then `gh auth setup-git`. |
| `cf` | `cf auth login --no-browser`. It shows a device code and a verify URL. |
| `wrangler` | `wrangler login --browser=false`. Its callback is `localhost:8976` on the machine: run `ssh -N -L 8976:localhost:8976 admin@<server>` on your client, then open the URL on the client. |
| Claude Code MCP with OAuth | `claude mcp add --scope user --transport http --callback-port 53682 <name> <url>`. Run `ssh -N -L 53682:localhost:53682 admin@<server>` on the client. In `claude`, run `/mcp`, choose the server, then open the link on the client. |
| Claude Code | `claude`, then `/login`. |
| Codex | `codex login --device-auth`. |
| treg | Give the machine its own agent token: `treg org agent-new <machine>`. Pipe the token to the machine and run `treg login --token` there. Do not print it. |

Close the `ssh -L` tunnels after the sign-in.

Set the git identity. Use the GitHub noreply address, which you can read with
`gh api user --jq '"\(.id)+\(.login)@users.noreply.github.com"'` on a signed-in
client:

```bash
git config --global user.name "Example Name"
git config --global user.email "12345+example@users.noreply.github.com"
```

## 6. Tokens for CLIs

Write CLI tokens from 1Password into a user-only file. Read each value inside
the pipe only:

```bash
{
  printf 'export EXAMPLE_API_TOKEN=%q\n' "$(op read 'op://Developer/<item-id>/credential')"
} | ssh admin@<server> 'umask 077; mkdir -p ~/.config/example-cli; cat > ~/.config/example-cli/env'
```

Load the file from `~/.bashrc`, before the `zoxide` line:

```bash
[ -f ~/.config/example-cli/env ] && . ~/.config/example-cli/env
```

Do not `grep` or `cat` shell files that contain secret values. List the
variable names only, for example `grep -o '^export [A-Z_]*' ~/.zshrc`.

See [1Password conventions](1password-conventions.md).

## 7. Work folders and skills

```bash
mkdir -p ~/work/projects ~/work/consulting
gh repo clone <owner>/skills ~/work/projects/skills
cd ~/work/projects/skills && ./install.sh
```

## 8. Check

```bash
node --version && bun --version && gh auth status && cf auth whoami && wrangler whoami
claude mcp list
rex server status
bash -ic 'z work && pwd'
```
