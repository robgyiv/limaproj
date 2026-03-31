# limaproj (Lima bootstrap)

This setup keeps your source on the host and runs builds/tests inside a per-project Lima VM.
It mounts only the active directory or repo the command is run from on the host into the VM at `/workspace`.

## Prereqs

- Lima installed and working (`limactl` on PATH)
- Python 3 available on the host

## Install

Add the wrapper to your PATH (fish):

```fish
set -gx PATH $HOME/.lima/_config/limaproj/bin $PATH
```

Reload:

```fish
source ~/.config/fish/config.fish
```

## Quick start (Apple silicon)

From inside your repo:

```bash
limaproj create proj-foo --policy untrusted

# Custom image
limaproj create proj-foo --policy untrusted \
  --image https://cloud.debian.org/cdimage/cloud/trixie/20260112-2355/debian-13-generic-arm64-20260112-2355.qcow2
```

Then:

```bash
limaproj start proj-foo
limaproj shell proj-foo
```

Inside the VM:

```bash
cd /workspace
```

## Policies

- **trusted**: repo mount + networking, installs build tools
- **untrusted**: repo mount + networking, minimal packages
- **airgap**: repo mount, no networking
- **devbox**: repo mount + networking, installs Docker CE (apt) + Nix package manager; provisions Python 3 + uv, Node.js LTS + pnpm, and LSP servers via Nix

"Networking" means outbound internet access from the VM. The repo mount (`/workspace`) is present on
all policies regardless — it is controlled by the `mounts:` key, not the network config.

`airgap` sets `networks: []` in Lima, removing all network interfaces including internet access.
`trusted`, `untrusted`, and `devbox` use Lima's default networking and have full outbound internet access.

The distinction between `trusted` and `untrusted` is intent and installed tooling only — use `untrusted`
when running unfamiliar third-party code you want isolated from your host filesystem but that still needs
internet access (e.g. `npm install` on an unknown repo).

Templates live in:

- `~/.lima/_config/limaproj/templates/base.yaml`
- `~/.lima/_config/limaproj/templates/trusted.yaml`
- `~/.lima/_config/limaproj/templates/untrusted.yaml`
- `~/.lima/_config/limaproj/templates/airgap.yaml`
- `~/.lima/_config/limaproj/templates/devbox.yaml`

## Neovim remote LSP (devbox policy)

The `devbox` policy provisions LSP servers inside the VM. You can point your local Neovim at them
over SSH so the LSP runs where the code is — no local Python or Node toolchain required.

### One-time setup

**1. Provision a devbox VM** (if you haven't already):

```bash
limaproj create my-devbox --policy devbox
limaproj start my-devbox
```

**2. Activate it as the Neovim target:**

```bash
limaproj use my-devbox
```

This writes `~/.ssh/conf.d/lima-devbox` with the VM's current SSH coordinates and ensures
`~/.ssh/config` includes it. Re-run `limaproj use` whenever you switch VMs or restart one
(the port changes on restart).

**3. Install the Neovim plugin file** (already written to `~/.config/nvim/lua/plugins/remote-lsp.lua`).
If your config uses LazyVim or a similar framework, it will be picked up automatically.

### Workflow

Each time you start a session:

```bash
limaproj start my-devbox   # if not already running
limaproj use my-devbox     # refresh SSH coords (port may have changed)
```

Then open Neovim on the host as normal — LSP commands (`gd`, `K`, diagnostics, etc.) work
transparently. The `ssh lima-devbox` invocation in each LSP `cmd` is what routes the stdio
protocol through to the VM.

**Switching projects:**

```bash
limaproj use other-devbox
```

All Neovim LSP clients share the single `lima-devbox` alias; switching it is enough.

### What's installed on the VM

| Tool | Installed via |
|---|---|
| `python3`, `uv` | Nix (`nixpkgs#python3`, `nixpkgs#uv`) |
| `node`, `pnpm` | Nix (`nixpkgs#nodejs`, `nixpkgs#pnpm`) |
| `pyright-langserver` | Nix (`nixpkgs#pyright`) |
| `typescript-language-server` | Nix (`nixpkgs#typescript-language-server`) |
| Docker CE | apt (needs systemd service management) |

All Nix-managed binaries land in `~/.nix-profile/bin/`, which is on `PATH` for login shells.

### Troubleshooting

**LSP not starting** — verify the SSH alias works:
```bash
ssh lima-devbox echo ok
```

**Port changed after restart** — run `limaproj use <name>` again to refresh `~/.ssh/conf.d/lima-devbox`.

**Slow LSP startup** — SSH connection setup on every LSP request can add latency. Add a `ControlMaster`
entry for `lima-devbox` to `~/.ssh/config` to reuse the connection:
```
Host lima-devbox
  ControlMaster auto
  ControlPath ~/.ssh/lima-devbox.sock
  ControlPersist 10m
```

## Notes

- The wrapper defaults to an amd64 Debian image; on Apple silicon, pass the arm64 image as above.
- You can stop and delete VMs with:

```bash
limaproj stop proj-foo
limaproj delete proj-foo
```
